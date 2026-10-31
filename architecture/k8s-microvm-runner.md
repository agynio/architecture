# k8s-microvm-runner

## Overview

The k8s-microvm-runner is a second implementation of the [Runner](runner.md) gRPC API. It runs every workload as its own Cloud Hypervisor microVM, hosted in a **pool of prewarmed slot Pods** on Kubernetes, and it can **hibernate** a workload — persist its memory and everything it wrote to its filesystems — and wake it later, in another slot, with its processes exactly where they were.

| Aspect | Detail |
|--------|--------|
| **Plane** | Data |
| **Language** | Go |
| **Repository** | `agynio/k8s-microvm-runner` |
| **API** | gRPC (`RunnerService`), including [`HibernateWorkload` / `WakeWorkload`](runner.md#workload-lifecycle) |
| **Backend** | Kubernetes API (slot Pods), Cloud Hypervisor (one microVM per workload) with virtiofsd (one filesystem share per microVM), the Kata Containers guest agent inside each microVM, an S3-compatible snapshot store |
| **Authentication** | OpenZiti network identity (SDK-embedded, service token enrollment) — as every runner |

It exists because the [k8s-runner](k8s-runner.md) ties a workload's lifetime to a Pod's. A Pod is either running and paying for its flavor, or gone — there is no state in between, so an agent that answers a message every few hours either idles between turns at full cost or is stopped and cold-started, with its processes lost, for every one of them. The k8s-microvm-runner separates the **compute host** (a slot) from the **workload** (a microVM's state). A workload that has nothing to do is a snapshot in object storage, costing storage and nothing else, and a slot is free the moment it leaves.

Isolation is the same as the k8s-runner's `kata-fc` option: a kernel per workload, hardware-enforced. What is new is that the VM is the platform's to pause, persist, move and resume.

## Terms

| Term | Meaning |
|---|---|
| **Slot** | An unprivileged Pod running the slot supervisor. Hosts at most one guest at a time; owns one VMM process, one network tap, and a local cache. Generic — no slot is tied to a workload, an environment, or an organization |
| **Guest** | The microVM a workload runs in: the Kata guest kernel and image, the Kata guest agent as init, and the workload's containers inside it |
| **Share** | The one virtio-fs filesystem a guest sees besides its own image: a directory tree on the slot's disk, assembled by the supervisor and served by virtiofsd, holding every container's root filesystem and every volume |
| **Template** | A snapshot of a guest booted for an environment and frozen before anything workload-specific exists. The starting point for a new workload in that environment |
| **Snapshot** | A guest's memory and VM state taken while paused, together with the deltas of its share. What hibernation stores |
| **Delta** | What a guest wrote to a filesystem — a container's writable layer, a volume's contents — kept apart from the immutable image it started from |
| **Snapshot store** | S3-compatible object storage holding templates, snapshots and volume deltas |

## Runner Catalog

Like the k8s-runner, the k8s-microvm-runner declares its [catalog](runners.md#runner-catalog) in its deployment configuration and reports it with `ReportRunnerCatalog`. A flavor here is a VM size, and `hibernate` is advertised as a capability so the [Agents Orchestrator](agents-orchestrator.md#hibernation) knows it may ask:

```yaml
catalog:
  flavors:
    - name: small
      default: true
      vcpus: 1
      memory: "2Gi"
      warm: 4          # idle slots kept ready for this flavor
      max: 50          # slots this flavor may occupy at once
    - name: large
      vcpus: 4
      memory: "16Gi"
      warm: 1
      max: 10
  storageClasses:
    - name: standard
      default: true
      # object-store bucket or tier for this class's deltas and snapshots
  capabilities: [docker, hibernate]
```

A flavor sizes the guest — the whole workload shares it, as it shares a Pod on the k8s-runner — and the slot Pod that hosts the guest is sized from it (the VM's memory plus the supervisor's fixed overhead). There is no per-container budget inside the guest; the kernel in the guest arbitrates as a kernel does.

`docker` is implemented the way the k8s-runner's `kata-fc` option implements it: a standard privileged DinD container, privileged relative to the guest kernel, not the host. `hibernate` is what this document is about.

## Workload-to-VM Mapping

| Workload Concept | k8s-microvm-runner realization |
|------------------|------------------------|
| Workload | One guest (microVM) in one slot |
| Main container | OCI container inside the guest, created and started through the guest agent |
| Sidecar | OCI container inside the guest, sharing the guest's single network namespace — `localhost:<port>` reaches an [MCP sidecar](mcp.md) exactly as in a Pod, and the Ziti sidecar's TPROXY covers the whole guest |
| Init container | OCI container inside the guest, run to completion before the main container, writing into the shared `agyn` volume |
| Ephemeral volume (`persistent: false`) | A directory in the share, discarded at stop. Part of the snapshot while the workload lives — a hibernated workload keeps its scratch space |
| Persistent volume (`persistent: true`) | A directory in the share whose contents are the **delta** kept in the snapshot store under the volume's key. The delta is the volume |
| Image | OCI layers pulled by the supervisor through the [image proxy](image-proxy.md) with the workload's pull credential, unpacked once per image digest into a directory tree that is the read-only lower layer of every container running that image |
| Image pull credential | Used by the supervisor, never by a node. There is no `dockerconfigjson` Secret and no `imagePullSecrets` |
| [Inline file](runner.md#inline-files) | Written into the guest by the supervisor before the container that mounts it starts |
| Labels | Kept in the runner's own workload index; `workload_key` and `volume_key` are what `ListWorkloads` and `ListVolumes` return, as on the k8s-runner |
| Flavor | VM size — vCPUs and memory |

Everything the platform assumes about a Pod — a shared network namespace, sidecars on localhost, one Ziti identity for the lot, a DinD daemon next to the agent — holds inside the guest, because the guest *is* the pod. The Orchestrator's spec is consumed unchanged.

## Slots

### Why a pool, and why Pods

Kubernetes cannot start anything in milliseconds: scheduling, CNI, image checks and container creation put a floor of seconds under every Pod. So a workload never waits for a Pod. The runner keeps `warm` idle slots per flavor ahead of demand, binds a workload to one that already exists, and replaces the slot it consumed in the background. The Pod cost is paid before anyone is waiting.

They are Pods rather than a node daemon so that the cluster keeps accounting for them: a slot is a scheduled unit of memory that preemption, eviction and the [workload PriorityClass](k8s-runner.md#workload-priority) treat like any other workload, and a node full of slots is visibly full. A daemon reserving memory the scheduler cannot see would be either over-committed or idle.

A slot is **generic**. It is not reserved for a workload — a workload idle for a month would otherwise hold a slot for a month, which is the cost this runner exists to remove. What persists while a workload is idle is its snapshot, and any slot of the right flavor can wake it.

### Slot Pod

A slot is deliberately an unprivileged Pod:

| | |
|--|--|
| **Devices** | `/dev/kvm` and `/dev/net/tun`, granted through a device plugin installed with the runner — the way KubeVirt grants them — as extended resources the slot requests. No `privileged: true`, no hostPath for devices. The guest's vsock is Cloud Hypervisor's own, in process, so no vhost device is needed |
| **Capabilities** | `NET_ADMIN` and `NET_RAW` for the tap and the NAT rule inside the Pod's own network namespace; `SYS_ADMIN` for the overlay and bind mounts the supervisor assembles the share from, inside the Pod's own mount namespace; `SYS_PTRACE`, which the kernel demands of a process that serves page faults for another address space, as the VMM does for a guest whose memory it restores on demand (the node sysctl `vm.unprivileged_userfaultfd=1` is the alternative); the file-ownership capabilities (`CHOWN`, `DAC_OVERRIDE`, `DAC_READ_SEARCH`, `FOWNER`, `FSETID`, `MKNOD`, `SETUID`, `SETGID`, `SETFCAP`) that unpacking an image and serving it as any user, by file handle, need, and `SETPCAP`, with which virtiofsd narrows its own set to those at start. Everything else is dropped. Running the slot Pod in a user namespace (`hostUsers: false`) confines every one of these to that namespace, so that `SYS_ADMIN` in particular is not the node's; that is the intended configuration wherever the cluster supports it, with the userfaultfd sysctl in place of `SYS_PTRACE`, which a user namespace cannot satisfy |
| **Sysctl** | `net.ipv4.ip_forward=1` as a Pod sysctl, which the kubelet must allow (`allowed-unsafe-sysctls`) — one more install-time requirement alongside the device plugin |
| **AppArmor** | `Unconfined`. The container runtime's default profile denies `mount` regardless of capabilities, and the share is made of mounts. The Pod's other confinement — its namespaces, its capability set, the VMM's seccomp filter — is what stands in for it |
| **Service account** | `automountServiceAccountToken: false`. A slot never talks to the Kubernetes API |
| **Resources** | Memory limit = flavor memory + supervisor overhead; CPU request from the flavor's vCPUs. The guest cannot exceed what the Pod was scheduled for |
| **Priority** | The [`agyn-workload` PriorityClass](k8s-runner.md#workload-priority), for the same reason the k8s-runner's Pods carry it |
| **Labels** | `agyn.dev/managed-by: agents-orchestrator`, so the [workload egress NetworkPolicy](k8s-runner.md#workload-egress-networkpolicy) applies to every slot unchanged — the guest's traffic leaves through the Pod's interface and is subject to it |
| **Storage** | A local cache volume (emptyDir, sized in the slot spec) holding base images, templates and the deltas of whatever the slot is hosting. Slots are long-lived, so a slot that keeps hosting one environment keeps that environment hot. A hostPath-backed cache shared by every slot on a node is an operator option for clusters whose Pod Security policy permits it; the runner works without it |

The Pod is the jail. One Pod hosts one VM, so Cloud Hypervisor and virtiofsd run as ordinary processes under the supervisor, with the VMM's seccomp filter on and virtiofsd confined to the share directory; the chroot and cgroup a VMM jailer would build are the Pod's own.

### Lifecycle

```
created ──▶ warming ──▶ idle ──▶ bound ──▶ idle ──▶ … ──▶ retired
```

| State | Meaning |
|---|---|
| `warming` | Pod scheduled, supervisor starting, caches being primed with the templates most recently used on this flavor |
| `idle` | Ready to host. Counts toward `warm` |
| `bound` | Hosting one workload — booting it, running it, or hibernating it |
| `retired` | Being deleted: it hit its reuse limit, the runner rolled out a new supervisor, VMM, virtiofsd or guest kernel, or it failed. Replaced, never repaired |

The runner creates slot Pods itself rather than through a Deployment, because the thing a Deployment cannot know — which slots are idle and which are bound — is the thing the runner must decide on. On startup it adopts existing slots by label, asks each supervisor what it holds, and rebuilds its index from that; a slot whose supervisor cannot be reached is retired.

When a workload leaves a slot — stopped or hibernated — the supervisor tears down the VMM and its virtiofsd, unmounts the share, deletes every per-workload file from the cache except what a wake would want again (that workload's own snapshot and deltas, kept for a bounded time), resets the tap, and reports `idle`. Nothing of one workload is reachable from the next: the next guest is a fresh VM whose only filesystems are the image and the share the supervisor hands it.

## Slot Supervisor

One process per slot, started as the Pod's command. The runner dials it over the Pod network on a channel it authenticates with a token it injected when it created the Pod; the supervisor dials nothing but the image proxy and the snapshot store. It holds no platform identity — the runner's OpenZiti identity is the only one in the deployment, and the supervisor is inside the runner's trust boundary, not the platform's.

It owns everything that happens inside the slot:

| Responsibility | Detail |
|---|---|
| **VMM** | Spawns Cloud Hypervisor, configures it over its API socket, pauses and resumes it, writes and restores snapshots, kills it. The runner ships its own Cloud Hypervisor, virtiofsd, guest kernel and guest image, pinned together; a snapshot records the versions it was taken with and is only restored by a supervisor running the same |
| **Network** | One tap per slot with a fixed MAC, a fixed point-to-point address pair that is the same in every slot, and a NAT rule to the Pod's own address. The guest's addressing, ARP cache and open connections are therefore valid in whichever slot it is restored into — the only thing that differs between slots is the Pod address, which the guest never sees |
| **Share** | Assembles the one directory tree the guest sees over virtio-fs, in the Pod's own mount namespace: for every container an overlay whose lower layer is the unpacked image and whose upper layer is the workload's writable layer, and for every volume a directory; then starts one virtiofsd serving the tree. The guest runs each container on its overlay directly — the host kernel merges the layers, so a container's writes cost slot disk, not guest memory, and the upper layer *is* the delta: a hibernation archives upper layers and volume directories, never images. The same image unpacks to the same path on every slot and the share has the same layout everywhere, so a woken guest's open files are re-opened by path by a fresh virtiofsd (its `find-paths` migration mode) in whichever slot the guest lands |
| **Guest** | Drives the Kata guest agent over vsock: creates the sandbox with the share mounted, creates and starts containers from the workload spec's OCI configuration with their root filesystems in the share, execs, streams container output, reseeds the guest's random pool and sets the guest clock after a wake. The guest agent is upstream Kata's, unmodified; the Kata shim is not involved |
| **Templates** | Builds the template for an environment the first time a workload of that environment lands on it, and publishes it to the store for every other slot |

## Guest

### Kernel, image and agent

The guest runs the stock Kata guest kernel and guest image, unmodified. The image is a virtio-pmem device mounted read-only with DAX, so its file data is the host's page cache, mapped into the guest rather than copied into guest memory; one copy on a node serves every guest on it, and none of it is in a snapshot. The guest agent is Kata's, started as the guest's PID 1 — there is no init system in the guest, nothing to boot but the agent, and nothing in a snapshot but the agent and the workload. The container runtime inside a microVM is a solved problem and the platform has no opinion on it; what the platform owns is the supervisor that drives it.

### Containers

A workload's containers run inside the guest as OCI containers with the guest's single network namespace in common. Images are not pulled inside the guest: each container's root filesystem is a directory in the share — the image's layers below, the container's writable layer on top, merged by the host — and the agent binds it into place. Init containers run first and populate the shared `agyn` volume, itself a directory in the share; the main container and sidecars start once they complete, as in a Pod. Each container also gets the sandbox's `/etc/hosts`, `/etc/resolv.conf` and `/etc/hostname` as a kubelet provides them: one writable file per sandbox in the share, bound into every container, so what one container writes — the Ziti enrollment rewrites `resolv.conf` — every other container sees.

### Clock and randomness

A restored guest resumes with the wall clock it was paused at. Cloud Hypervisor advances the guest's clock by the downtime when it restores a snapshot, and the supervisor sets the guest clock from the host on top of that, before anything in the guest is signalled. Monotonic time does not jump — a timer that had ten seconds left still has ten seconds left — which is what lets processes carry on unaware; what they observe is that the wall clock moved while they were not looking, the same thing a laptop's processes observe after a lid is closed and opened.

A restored guest also resumes with the kernel's random pool exactly as it was frozen, and two guests restored from one template would share it until the kernel's next scheduled reseed — up to a minute — and hand out the same keys, nonces and session ids. Cloud Hypervisor has no VM generation identifier device to tell the guest it was cloned, so the supervisor does: immediately after resume, before the clock is set and anything is signalled, it feeds fresh entropy into the guest's pool through the guest agent, which mixes it in and reseeds. Templates are additionally frozen *before* any workload process has run, so no user-space random state — a TLS session, a generated key, a seeded PRNG — exists in a template to be shared.

### Networking and egress

The guest has one interface with a fixed address and a default route to the supervisor's tap. The supervisor NATs it to the Pod. The tap and the guest interface carry the Pod interface's MTU — 1450 on a VXLAN overlay, not the 1500 a guest assumes — because the NAT between them relays no path-MTU discovery, and a guest sending full-size segments stalls on every large transfer while small requests keep working, which looks like a slow platform rather than a network fault. From there it is the Pod's traffic: the workload namespace's egress NetworkPolicy applies, the Ziti sidecar inside the guest reaches the overlay as it does from a Pod, and nothing in the guest can reach the supervisor's API or the Pod's own network namespace beyond the tap.

The supervisor's own traffic — image pulls and snapshot transfers — is Pod traffic under the same policy, which is why the policy needs the two rules listed under [Namespace and RBAC](#namespace-and-rbac). It pulls by the hostname in the image reference, as the node's runtime does on the k8s-runner: the runner holds no registry address, the workload spec carries none, and making the proxy's hostname resolve from a Pod to an address the policy admits is the deployment's job — an ingress the name already resolves to needs nothing; the [local bundle](operations/local-bundle.md), whose name resolves to the node's own loopback, rewrites it in cluster DNS. Where the proxy presents a certificate from a private CA, the runner is configured with that CA and mounts it into every slot Pod, and the supervisor trusts it in addition to the system roots. Nothing else about the pull differs from any registry client's.

## Workload Lifecycle

### Start

```mermaid
sequenceDiagram
    participant O as Orchestrator
    participant R as k8s-microvm-runner
    participant S as Slot supervisor
    participant G as Guest

    O->>R: StartWorkload(spec)
    R->>R: Resolve flavor; take an idle slot of that flavor
    R->>S: Bind(spec)
    alt template for this environment in cache or store
        S->>S: Restore template (memory + share)
    else no template yet
        S->>G: Boot guest; pull images via image proxy; create and start containers, gated
        S->>S: Snapshot as template; publish
    end
    S->>G: Deliver per-workload values; set clock; release gates
    G->>G: Ziti sidecar enrolls; agynd starts
    S-->>R: running
    R-->>O: instance_id
    R->>R: Create a replacement slot in the background
```

A **template** is a guest in which every container of the environment has been created and started, but each container's entrypoint is held at a gate until the workload it belongs to is known. Everything that is the same for every workload of the environment — the images, the init containers' output, the MCP sidecars' processes up to their first read of their port, the mounts — is in the template. Everything that is not — the workload and instance ids, the enrollment JWT, resolved ENVs and secrets, inline files, the pull credential — is delivered at bind time into the guest and read by the gates, which then exec the real entrypoints. The template is keyed by a digest of everything in the spec that is not per-workload — every container's resolved image digest, command, mounts, ports and plain environment, the flavor — together with the supervisor's own version set, so any change to what the environment contains produces a new template and never a stale one.

Binding to a template is what makes starting a new workload a matter of milliseconds plus whatever the workload's own processes take to come up. The first workload of an environment on a runner pays for building the template; no other does.

A workload whose environment has no template yet on this runner, or whose template build fails, boots cold through the same path without the snapshot step. Cold boot is seconds, not the tens of seconds a Pod takes — there is no scheduler, no CNI and no kubelet in the way — but it is the exception the pool exists to avoid.

### Hibernate

```mermaid
sequenceDiagram
    participant O as Orchestrator
    participant R as k8s-microvm-runner
    participant S as Slot supervisor
    participant ST as Snapshot store

    O->>R: HibernateWorkload(workloadId)
    R->>S: Hibernate
    S->>S: Pause VM; snapshot memory and VM state; archive the share's deltas
    S->>ST: Upload memory, state and deltas under the workload's key
    S->>S: Kill VMM and virtiofsd; keep a cached copy; clear the slot
    S-->>R: hibernated (snapshot size)
    R-->>O: OK
    R->>R: Slot returns to idle
```

The guest is paused for the duration of the memory snapshot — under a second per gigabyte of populated guest memory on local disk. Everything the guest can see goes with it: the memory, the state of every device, and every container's writable layer and every volume, scratch space included. A workload mid-build, with a shell open and a process waiting on a socket, is preserved as such.

The memory snapshot is **sparse**. Guest memory is a shared memory file, and the guest reports the pages it frees back to the VMM (free page reporting), so a page the workload never touched or has released is a hole in the file, not a page of zeros; the snapshot is the size of what the guest actually holds, not of its flavor. After a wake that mapped memory on demand, the next snapshot holds only the pages touched since, and the supervisor merges it over the one it woke from before uploading, so the store holds one whole snapshot per workload and a wake never replays a chain. A workload hibernated after every turn therefore pays, per turn, a snapshot of what it touched and an upload of what it holds.

The snapshot holds the workload's secrets and memory. The store is configured with server-side encryption and the runner's credentials are the only ones that can read it.

A hibernation whose upload fails does not fail the workload: the snapshot is whole on the slot's disk, so the supervisor restores the guest from it and the workload keeps running where it was, the partial objects are removed from the store, and `HibernateWorkload` returns the error. `failed` is reported only when that restore fails too.

### Wake

```mermaid
sequenceDiagram
    participant O as Orchestrator
    participant R as k8s-microvm-runner
    participant S as Slot supervisor
    participant ST as Snapshot store
    participant G as Guest

    O->>R: WakeWorkload(workloadId, image_pull_credentials)
    R->>R: Prefer the slot still caching this snapshot; else any idle slot of the flavor
    R->>S: Wake
    S->>ST: Fetch VM state and deltas; map memory for on-demand fetch
    S->>S: Rebuild the share at the same paths; start virtiofsd
    S->>G: Restore snapshot; resume
    S->>G: Reseed random pool; set clock; SIGCONT to each container's init
    S-->>R: running
    R-->>O: OK
```

Restoring a snapshot and resuming the VM is a matter of milliseconds. What then costs is **population**: a restored guest's memory is mapped on demand, and every page the workload touches for the first time after a wake is a fault the VMM serves from the memory file in the slot's cache. A wake into a slot that does not hold the snapshot first fetches the memory image and the deltas from the store and then restores on demand from that local copy — the VMM restores from local files, not from the store page by page. The runner prefers the slot that last hosted the workload for exactly this reason, and a wake into it after a short hibernation fetches nothing and touches nothing that is not already local. The share is rebuilt before the guest resumes — the images re-unpacked if the slot lacks them, the deltas put back, the overlays mounted at the paths the guest knew — and the fresh virtiofsd hands the guest back its open files by path.

A woken guest's network is valid immediately — same address, same MAC on the other end of the tap — so a connection that was open stays open as far as the guest can tell. What has actually lapsed on the far side (a Gateway stream, an edge router session) is reconnected by the processes that own it: the supervisor sends `SIGCONT` to the init process of every container once the clock is set, which a process that has no handler for it never sees and [`agynd`](agynd-cli.md#8-wake) takes as its cue to resubscribe. The Ziti sidecar re-establishes its sessions on its own. The supervisor signals once the clock is set and once more a few seconds later: in the prototype a signal sent in the first half second after resume never reached its handler, and the repeat is free because the handler is idempotent. `agynd` also recognises a wake on its own — the wall clock jumping ahead of monotonic time between two of its ticks, which is what the clock set after a restore looks like from inside — so a signal lost twice costs one tick, not a stuck workload.

### Stop and Remove

`StopWorkload` on a running workload archives each persistent volume's directory under its key, so the delta in the store is current, then kills the guest and discards its writable layers, scratch volumes and memory; on a hibernated workload it deletes the snapshot from the store. Neither deletes a persistent volume's delta — it is the volume, and `RemoveVolume` is what deletes it. `RemoveWorkload` is a stop followed, when asked, by the volumes' removal.

### Exec, logs, inspect

`Exec` reaches a process inside the guest through the guest agent, with TTY, resize and the timeouts the contract requires — the [Terminal Proxy](terminal-proxy.md) and [workspace sync](sandbox-sync.md) work unchanged. `StreamWorkloadLogs` streams what the supervisor captured from a container's stdio; the capture travels with the workload's deltas, so a workload woken in another slot still has its logs, and it is discarded with a stop, as Pod logs are. `InspectWorkload` answers from the runner's index and the supervisor's view of the guest: per-container status, exit codes and restart counts come from the guest agent. A hibernated workload reports `hibernated` with the container list it had when it was paused.

## Volumes

A persistent volume on this runner is **not a PersistentVolumeClaim**. A PVC binds when a Pod is created and cannot be attached to a Pod that already exists, which is every slot; and a PVC pins its workload to a node and a storage backend that a wake elsewhere cannot follow. Instead a volume is a directory in the share whose contents the supervisor persists:

| | |
|---|---|
| **Provisioned** | On first `StartWorkload` that names its `volume_key`: an empty delta is recorded in the store under that key. `ListVolumes` returns it with the key, as the k8s-runner returns PVCs |
| **Mounted** | A directory the supervisor binds into the share, bound into the container at the declared path by the guest agent. An MCP sidecar's [`shared_volumes`](resource-definitions.md#mcp) are the same directory bound into two containers. The declared size is a project quota on the directory where the cache filesystem supports one (`prjquota`), and advisory where it does not, which the supervisor logs once per volume |
| **While hibernated** | The delta is in the store — the directory's contents, archived at hibernation under the volume's key. A hibernated workload's filesystems are not a separate artifact from its volumes |
| **Removed** | `RemoveVolume` deletes the delta. The [Agents Orchestrator](agents-orchestrator.md#volume-reconciliation) decides when, exactly as for PVCs |
| **Storage class** | Names the store tier (bucket, class, retention policy) deltas for that volume are written to |

A workload therefore has no affinity to a node at all. It can wake on any slot of its flavor in the cluster, and the question "which runner holds this instance's volumes" has the same answer as on the k8s-runner — the runner — without the node pinning underneath it.

## Snapshot Store

An S3-compatible bucket the runner is configured with, holding three kinds of object under separate prefixes:

| Object | Keyed by | Lifetime |
|---|---|---|
| Template | Environment image digests + flavor + supervisor version set | Until unreferenced for a configured period, or the version set it was taken with is retired |
| Workload snapshot | `workload_key` | Until `StopWorkload` / `RemoveWorkload` discards it |
| Volume delta | `volume_key` | Until `RemoveVolume` |

The runner reports a hibernated workload's snapshot size on its [state report](#workload-state-reporting) so the platform can meter it; what the store charges for the bytes is the operator's arrangement with the store.

## Workload State Reporting

The runner reports transitions through `ReportWorkloadState` on the [Runners Gateway](gateway.md), like the k8s-runner — observed from its supervisors rather than from a Pod informer. It reports `running` when a guest's containers are up, `hibernated` when a snapshot is durable (with its size), `failed` when a guest dies or a snapshot cannot be taken or restored, and gone when it no longer holds a workload. Its `ListWorkloads` answer carries every workload a supervisor holds, keyed by the `workload_key` the Orchestrator put in the start request's labels, and `InspectWorkload` lists every container of the spec from the moment the start is accepted — init containers as they complete, the main container and sidecars as waiting until they run — so the Orchestrator's reconciliation can match and classify a starting workload the same way it does a Pod. A workload missing from the list is one the Orchestrator treats as lost and revokes. The [Orchestrator](agents-orchestrator.md#workload-reconciliation) stays the owner of desired state and its reconciliation the backstop, exactly as [Runners — Runner-Reported Workload State](runners.md#runner-reported-workload-state) describes.

## Isolation

| Boundary | What enforces it |
|---|---|
| Workload ↔ host kernel | The microVM. The guest runs its own kernel; the host sees a KVM process |
| Workload ↔ other workloads | One guest per slot, one slot per Pod. Two workloads never share a kernel, a Pod, or a network namespace |
| Workload ↔ slot | The guest reaches the tap and, through virtiofsd, the share directory and nothing else. The supervisor's control channel, the VMM's API socket and virtiofsd's socket are on the Pod's filesystem, which the guest cannot see |
| Slot ↔ node | An unprivileged Pod: two devices, a handful of capabilities confined to the Pod's own namespaces, no host mounts, no API token |
| Slot reuse | A new guest is a new VM whose only inputs are the image and the share the supervisor serves it. Deltas and snapshots of a previous workload are files outside the next guest's share |
| Secrets at rest | A snapshot contains the workload's memory and resolved secrets. The store is encrypted at rest and readable only with the runner's credentials; the slot cache lives in Pod-local storage that goes with the Pod |

What the k8s-runner's `kata-fc` option gives — a kernel per workload, so a kernel vulnerability in one cannot reach another — this runner gives too, with the slot Pod's reduced privilege around it rather than Kata's privileged shim on the node.

## Namespace and RBAC

Slot Pods are created in the workload namespace alongside whatever the k8s-runner creates there, so the namespace's egress NetworkPolicy, Pod Security configuration and PriorityClass apply once. The runner's ServiceAccount needs less than the k8s-runner's: `pods` (`get`, `list`, `watch`, `create`, `delete`) and `events` (`get`, `list`, `watch`). No `pods/exec` — exec goes through the supervisor; no `secrets` — nothing is materialized as a Secret; no `persistentvolumeclaims` — there are none.

Install-time requirements beyond the k8s-runner's: the device plugin DaemonSet, the kubelet's `allowed-unsafe-sysctls` including `net.ipv4.ip_forward`, a Pod Security configuration for the workload namespace that admits the slot Pod's capability set and unconfined AppArmor profile, nodes with KVM (bare metal or nested virtualization enabled) and a kernel whose overlayfs the slot Pod may mount (any recent kernel; in a user namespace, 5.11 or later), the guest kernel's idle behaviour chosen for the node — under nested virtualization a halted vCPU wakes tens of milliseconds late, which turns every packet and every vsock message into a stall, so the runner boots guests with `idle=poll` there, at the price of a busy vCPU per guest; on hardware KVM the default idle path is right and polling would only burn CPU —, transparent huge pages allowed for shared memory on the node (`transparent_hugepage/shmem_enabled` at least `advise`; the VMM asks for them), because guest memory is a shared memory file and every first touch of a guest page is otherwise a stage-2 fault per 4 KB — on hardware a cost, under nested virtualization a disaster: 256 MB took 17 s to allocate, 63 s once free-page reporting had returned the pages, and 0.9 s with huge pages, the snapshot store with its credentials in the runner's configuration, two more egress rules in the workload NetworkPolicy — the image proxy as a Pod reaches it, and the snapshot store — and, where the proxy's certificate is not publicly trusted, the proxy's CA in the runner's configuration. On the k8s-runner the node pulls images from outside the Pod's policy, with the node's resolver and trust store; here the slot Pod itself pulls, so the proxy's hostname must resolve from a Pod to an address the policy admits, and the policy that blocks cluster-internal addresses would otherwise block the pull. For the same reason the [local bundle](operations/local-bundle.md)'s image bake, which fills the node's container store, does not pre-warm this runner: a slot's cache fills from the image proxy on first use, unless the bake is extended to fill it.

## What Hibernation Does Not Cover

- **Time passes.** A sleeping workload does not run. A cron inside it, a timer, a poll loop — none of them fire until a wake, and then they fire having missed everything in between. That is the contract: the platform wakes a workload when it has something for it, not on the workload's own schedule.
- **Far ends time out.** Connections the guest held are intact in the guest and dead on the other side. Platform-side processes reconnect on the wake signal; a workload's own long-lived connection to some third-party service may not, and software that assumes an uninterrupted socket is wrong on a laptop too.
- **A runner without the capability.** On the k8s-runner the [idle timeout](agents-orchestrator.md#idle-timeout) stops workloads as before. Hibernation is something a runner advertises, and the Orchestrator asks for it only where it is advertised.

## Classification

| Aspect | Detail |
|--------|--------|
| **Plane** | Data |
| **API** | gRPC (`RunnerService`) |
| **State** | Runner index of workloads, volumes and slots, rebuilt from the supervisors and the store on restart; templates, snapshots and deltas in the snapshot store. No database |
| **Scaling** | Single replica per cluster; capacity scales with the slot pools |
| **Failure impact** | Loss of the runner prevents starts, wakes, hibernations and exec; running guests continue in their slots, and hibernated workloads are safe in the store. Loss of a slot kills the guest in it — reported `failed`, restarted by the Orchestrator from its volumes; a workload hibernated in the store is unaffected by any slot's loss |
