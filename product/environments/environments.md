# Flavors and Environments

## Purpose

Flavors and environments separate "where and how big a workload runs" from "what the workload does."

- **Flavor** — a named compute size (CPU/memory) offered by a specific runner, declared in the runner's own configuration and reported by the runner to the platform. Flavors replace free-form per-agent CPU/memory configuration with a curated menu, the way cloud providers offer instance types.
- **Storage class** — a named storage tier offered by a specific runner, declared and reported the same way. Persistent [volumes](../../architecture/resource-definitions.md#volume) pick a class the way workloads pick a flavor.
- **Environment** — an organization-level runtime definition: a runner, a flavor on that runner, the [images](../images/images.md) a workload runs, and everything a workload in it contains — its volumes, MCP servers, init scripts, environment variables, and egress rules. Environments are the unit both [agents](../../architecture/resource-definitions.md#agent) and [sandboxes](../sandboxes/sandboxes.md) run in.

Before this feature, every agent carried its own inline `image`, `resources`, and `runner_labels`. That put capacity decisions in the hands of every agent author, made sizes inconsistent across an organization, and left nothing for a non-agent workload (a sandbox) to run as.

Storage followed the same path later. Volumes used to be free-standing organization resources attached to agents and MCP servers through a separate attachment record — an entity holding a mount path and a size, representing nothing that existed until something mounted it, and describing storage for a workload it had no relationship to. Volumes are now declared on the environment that mounts them, which is also what lets an agent and a sandbox in the same environment have the same disks laid out the same way.

## User Stories

- As a runner operator, I want to declare the compute sizes and storage tiers my runner offers in the runner's own deployment config, so the menu lives next to the implementation that honors it and is managed with the same GitOps tooling.
- As an organization owner running my own runner, I want to offer custom flavors on it — including sizes the shared runners don't offer.
- As an organization owner, I want to define an environment (image + size) once and have many agents — and engineers' sandboxes — run in it.
- As a platform admin, I want to apply platform resources and runner configuration in either order — an environment may name a flavor before the runner first reports it.
- As an operator, I want to attach egress rules and secret-backed environment variables to an environment, so everything running in it (agent or sandbox) gets the same network policy and credentials.
- As an operator, I want to declare where a workload's disks are mounted and how big they are on the environment, so an agent and a sandbox running it get the same layout — and so an environment that needs no persistence declares none.
- As an operator, I want an MCP server to read files the agent writes, without a separate resource standing between them.
- As an engineer, I want to author my own environment without being an organization owner, and to decide who else may run in it.
- As a platform operator, I want usage metered against the flavor a workload held, so billing can be priced per size tier.

## Concepts

| Term | Definition |
|---|---|
| **Runner catalog** | The set of flavors, storage classes, and capabilities a runner offers. Declared in the runner's deployment configuration and reported by the runner; not managed through platform APIs. |
| **Flavor** | A named compute size (requests/limits for CPU and memory) belonging to one runner's catalog. |
| **Storage class** | A named storage tier belonging to one runner's catalog. What backs it is the runner's concern. |
| **Environment** | An organization-scoped runtime definition: a runner + a flavor name + a workspace image and, optionally, an agent runtime image — plus the volumes, MCP servers, init scripts, ENVs, and egress rules a workload in it carries. Referenced by agents and sandboxes. |
| **Volume** | A named mount declared on an environment (or on an MCP server): a path, a size, and whether it persists. A definition, not a disk — one disk is provisioned per agent instance or sandbox that runs it. |

## The Runner Catalog

A flavor or storage class belongs to exactly one runner — deliberately. Runners exist to offer different capabilities: CPU architectures, devices, memory or storage types and amounts. A compute size or storage tier is only meaningful relative to the specific runner's infrastructure, so there is no cluster-wide catalog; "the same size on two runners" is two flavors.

The catalog is **declared in the runner's own deployment configuration and reported by the runner** (at startup and on config change) — not created through platform APIs. The runner is in charge of implementing every entry it offers, so the declaration lives next to the implementation: the k8s-runner's config maps each storage class to a Kubernetes StorageClass, and its flavors to pod resource requests/limits. Whoever operates the runner deployment — cluster operators for shared runners, organization teams for their own — controls the catalog through that config; the platform only reflects it:

| Runner scope | Where its catalog is managed | Who can use it |
|---|---|---|
| Cluster-scoped | The runner's deployment config (cluster operators) | Environments and volumes in any organization |
| Org-scoped | The runner's deployment config (the owning organization) | Environments and volumes in that organization only |

### Flavors

- Flavor names are unique per runner (e.g., `small`, `standard`, `large`).
- A flavor defines Kubernetes-style compute resources: `requests_cpu`, `requests_memory`, `limits_cpu`, `limits_memory`.
- At most one flavor per runner is marked `default` — used when an environment names no flavor.
- A flavor can be marked **deprecated** — a soft signal: pickers warn against new references, existing references keep scheduling.
- Removing a flavor from the runner's config removes it from the catalog; environments still naming it become unschedulable and are flagged.

### Storage Classes

- Storage class names are unique per runner (e.g., `standard`, `fast-ssd`); `default` and `deprecated` work exactly as for flavors.
- A persistent [volume](../../architecture/resource-definitions.md#volume) may name a storage class; the name is resolved on the runner the workload lands on when the volume is first provisioned. A volume that names none gets the runner's default class.
- The class applies at provisioning time only. Already-provisioned volumes keep their storage, whatever happens to the catalog.

## Environments

An environment is an org-scoped resource any member can author:

| Field | Meaning |
|---|---|
| `name` | Unique within the organization |
| `runner` | The runner workloads run on. Must be visible to the organization — validated at create/update |
| `flavor` | Name of a flavor in that runner's catalog. Not validated at create/update — resolved at every workload start. Empty means the runner's default flavor |
| `workspace image` | An [image](../images/images.md) of type `workspace`, plus a tag. Runs as the main container |
| `agent runtime image` | Optionally, an image of type `agent_runtime`, plus a tag. Runs as an init container and supplies the agent CLI |
| `availability` | `internal` — any org member may run workloads in it. `private` — only identities holding a role on it. See [Who Can Use an Environment](#who-can-use-an-environment) |

Both images are selected from the [image catalog](../images/images.md) — an environment holds no registry addresses, and neither does anything downstream of it.

### Why the agent runtime lives here

The agent CLI an agent runs is a property of the environment, not of the agent. Two agents wanting different CLIs on the same workspace image therefore need two environments, and that duplication is accepted: it keeps a single answer to "what does a workload in this environment contain," which is what lets a [sandbox](../sandboxes/sandboxes.md) started against an environment carry the same tooling an agent there would.

An environment may name **no** agent runtime image. That is a workspace-only environment — perfectly usable by sandboxes, and rejected by `CreateAgent`, which has no agent CLI to run without one.

What is *not* in the environment is `agynd` and the [`agyn`](../../architecture/agyn-cli.md) CLI. Each ships with the platform in its own init image and is injected into every workload; they are not a configuration surface. See [Agent Init Container](../../architecture/agent-init.md).

## What an Environment Contains

An environment answers "what does a workload in this environment contain," and it answers it the same way for an agent and for a sandbox:

| | Declared on the environment | Also declarable on an agent | How they combine |
|---|---|---|---|
| **Volumes** | ✓ | — | Environment only |
| **MCP servers** | ✓ | ✓ | Union, agent wins on name |
| **Init scripts** | ✓ | ✓ | Environment's run first, then the agent's |
| **ENV variables** | ✓ | ✓ | Union, agent wins on name |
| **Egress rules** | ✓ | ✓ | Union |

Everything in the first column applies to every workload running the environment — agent workloads and [sandboxes](../sandboxes/sandboxes.md) alike. It is how sandboxes get network policy and credentials at all: they have no agent to attach anything to. It is also why a sandbox is a genuine copy of an agent's runtime rather than an approximation — the same sidecars, the same init, the same disks.

Registry credentials are not among them. They belong to the [image](../images/images.md), are held by the platform, and are never delivered to a workload or its cluster — see [Images — How Images Are Pulled](../images/images.md#how-images-are-pulled).

### Volumes

A volume declares a mount: a name, a path, whether it persists, and — when it does — a size, a storage class, and an optional TTL.

```
environment "dev"
  volume "workspace"  /workspace   persistent, 10Gi
  volume "cache"      /cache       ephemeral
```

**A volume is a definition, not a disk.** Each agent instance and each sandbox running `dev` gets its own `/workspace` — same layout, separate storage. Nothing is shared between owners; two agents in one environment cannot see each other's files, and neither can two engineers' sandboxes. Sharing across workloads is not part of the volume model, and an operator who needs it reaches for external storage or an MCP server that fronts it.

**There are no default volumes.** An environment declaring none runs workloads whose writes all land on the container's own ephemeral disk and vanish when the workload stops. That is a legitimate configuration — a stateless agent needs nothing else — and it is the reason persistence is declared rather than assumed. What it also means is worth stating plainly: [agent state](../../architecture/agent/state.md) survives a restart only in environments that provide for it, and a [sandbox](../sandboxes/sandboxes.md) in such an environment comes back empty after an idle stop.

An MCP server can declare volumes of its own the same way — private to that sidecar.

#### Sharing a volume with an MCP server

The one case where two containers need the same files is an agent writing something an MCP server reads. An MCP names the environment volumes it wants:

```
mcp "indexer"   shared_volumes: ["workspace"]
```

and mounts them where the main container has them. One disk, two containers. The reference is by **name**, resolved when the workload starts against whichever environment it is running in — so `dev`, `staging`, and `gpu` can each declare a `workspace`, and an agent-level MCP asking for one works in all three. Repointing an agent to another environment needs no edit anywhere. A name that resolves to nothing fails scheduling and flags the environment, the same treatment an unknown flavor gets.

Names are unique within an environment, which is what makes them referenceable, and deliberately reusable across environments, which is what makes environments substitutable.

### MCP servers and init scripts

Both can be declared on the environment or on an agent. The distinction is about *reach*, not about kind:

- **On the environment** — every workload running it gets them, including sandboxes. This is where tooling common to a whole runtime belongs: the MCP server every agent on this image needs, the init script that configures the toolchain.
- **On the agent** — only that agent's workloads. This is where an agent's own tools and setup belong.

An agent-level MCP with the same name as an environment-level one replaces it; init scripts never collide — the environment's run first, then the agent's, each in creation order.

## Who Can Use an Environment

Everything above — the volumes' contents, the secret-backed ENVs, the egress rules that inject credentials — is reachable from a shell by anyone who can start a [sandbox](../sandboxes/sandboxes.md) in the environment. Environments are the intentional sharing boundary, so **using** an environment is a permission of its own, separate from editing it:

| Role | Can |
|---|---|
| `owner` | Manage roles, change availability, delete, edit configuration, run workloads |
| `maintainer` | Edit configuration, run workloads |
| `user` | Run workloads — start a sandbox, or point an agent at it. No configuration access |

Any organization member may create an environment, and the creator becomes its `owner`. `availability` sets the default reach: `internal` lets any org member run in it, `private` restricts that to role holders. Either way every member can *see* the environment's name, runner, flavor, and images — pickers stay populated — while its volumes, ENVs, MCPs, and init scripts are visible only to those who can read its configuration.

Organization owners retain administrative access to every environment in the organization.

Credentials that only one team should hold belong in a `private` environment with `user` granted to that team, or in agent-level attachments — not in an `internal` environment.

## Placement

An environment references its runner directly — **the environment fully determines placement**. The flavor name selects a size within that runner's catalog, late-bound at workload start:

1. **Environment create/update** — the runner must be visible to the organization (cluster-scoped, or org-scoped to the same org); a wrong runner pairing is rejected here. The flavor name is deliberately **not** validated — an environment may name a flavor the runner hasn't reported yet, so platform resources and runner config can be applied in either order. Console and CLI warn (not reject) when the name doesn't match anything currently reported — the guard against typos.
2. **Workload start** — the [Agents Orchestrator](../../architecture/agents-orchestrator.md) schedules the workload only on the environment's runner, resolving the flavor name (and every volume's storage class name) against the runner's reported catalog. There is no fallback to another runner — a different runner has no contract to honor the names. If the runner is unavailable or a name doesn't resolve, the standard start retry policy applies and the failure is surfaced on the workload.
3. **Unschedulable flagging** — removing a runner that still backs environments, or removing a flavor from a runner's config while environments still name it, leaves those environments (and the agents and sandboxes referencing them) unschedulable. They are flagged in the Console and CLI rather than silently rescheduled.

Organizations that ran the same agent on different hardware tiers (e.g., staging vs. production runners) express that as two environments.

## Agent Changes

- Agents reference an environment (`environment_id`) instead of carrying inline `image`, `resources`, and `runner_labels`. The orchestrator resolves environment → images + runner + flavor at workload spec assembly. An agent must name an environment that has an agent runtime image.
- `runner_labels` are removed from the agent: placement intent now lives entirely in the environment. Agent `capabilities` remain — the environment's runner must advertise (report) every capability the agent requires, checked at workload start; Console and CLI warn at agent create/update when the runner's current report doesn't cover them.
- Agent-level egress rule attachments, ENVs, MCP servers, init scripts, and image pull secret attachments continue to work and compose with the environment's (union; agent-level ENV and MCP win on name conflict).
- Agents no longer carry storage of any kind. Volumes are declared on the environment, and pointing an agent at an environment is what gives it disks.
- MCP sidecars keep their own `resources` — adopting flavors for sidecars is out of scope. Their images come from the [catalog](../images/images.md) like everything else.

### Migration

Flavor sets are defined per runner before migration. Each existing agent is migrated by generating one environment per distinct `(image, resources)` pair in its organization, mapped to the smallest flavor on the agent's current runner that covers its requests. Inline fields are removed afterward.

Volumes are not migrated. The old free-standing volume definitions and their attachments are dropped, and operators redeclare the storage they want on the environments that need it — a rewrite small enough that carrying a translation of a model that described nothing would cost more than redoing it.

## Metering

Compute is metered per flavor. A workload emits `FLAVOR_SECONDS` dimensioned by the flavor it occupies and the runner whose catalog declares it, so usage is priced per tier the way cloud billing is priced per instance type. CPU and memory are two numbers inside a flavor; billing them separately re-derives a shape the platform already names, in units nobody prices.

The flavor billed is the one recorded on the workload when it started, not the one its environment names now — repointing an environment or editing a catalog entry does not rewrite history.

Every started workload carries a flavor, so there is no unmetered compute. Agents reference an environment and sandboxes run one, and a flavor name that does not resolve against the runner's catalog leaves the environment [unschedulable](#placement) rather than starting a workload without one.

**Next phase:** storage moves the same way — dimensioned by `storage_class`/`runner_id` rather than raw `GB_SECONDS`.

## Lifecycle

| Event | Effect |
|---|---|
| Runner reports its catalog | Reported flavors and storage classes become resolvable immediately (per runner visibility). Environments that already named them start scheduling. |
| Flavor/class added to runner config | In the catalog on the next report. |
| Flavor/class changed in runner config | New values apply to workloads started (volumes provisioned) after the next report. Running workloads are not resized; existing volumes keep their storage. |
| Flavor/class marked deprecated | Soft signal: pickers warn against new references. Existing references keep scheduling. |
| Flavor/class removed from runner config | Gone from the catalog on the next report. Environments/volumes still naming it become unschedulable and are flagged. Renames are a removal plus an addition. |
| Environment created | Available to agents and sandboxes in the organization — even if its flavor name isn't reported yet (it won't schedule until it is). |
| Environment updated (images, runner, or flavor) | Applies to workloads started after the change. Running workloads are not restarted. |
| Volume added to an environment | Mounted by workloads started after the change; provisioned per owner on first use. |
| Volume removed from an environment | The mount disappears from workloads started after the change, and every disk provisioned from it is deprovisioned across all owners. |
| Volume's size or storage class changed | Applies to disks provisioned after the change. Existing disks are neither resized nor migrated. |
| MCP or init script added to or removed from an environment | Applies to workloads started after the change, agent and sandbox alike. |
| Environment deleted | Fails while any agent or sandbox references it. |
| Runner removed | Environments referencing it become unschedulable and are flagged. |
| Referenced image deleted, tag gone upstream, or visibility narrowed | Environments naming it become unschedulable and are flagged — the same treatment as a removed flavor. See [Images — Lifecycle](../images/images.md#lifecycle). |

## Constraints

- A flavor or storage class belongs to exactly one runner's catalog; there is no cross-runner identity. Two runners offering "the same" size are two flavors, and references are meaningful only relative to the referenced runner.
- Catalog entries are referenced by name and resolved at workload start / volume provisioning — never by stored ID, never validated at resource create time.
- An environment references exactly one runner, at most one flavor name, one workspace image, and at most one agent runtime image.
- Running workloads never pick up catalog or environment changes in place — changes apply on next workload start. Provisioned volumes never change storage class.
- A volume belongs to exactly one environment or one MCP server. There is no free-standing volume resource and no attachment record.
- A volume name is unique within its environment and reusable across environments. Mount paths are unique within a target, and within any single container after shared volumes are resolved.
- One disk per definition per owner. No storage is shared between agent instances, between sandboxes, or between an agent and a sandbox — only between containers of one workload.
- Nothing is provisioned by default. An environment with no volumes produces workloads with no persistent storage.

## Related Architecture

- [Images](../images/images.md)
- [Agent Init Container](../../architecture/agent-init.md)
- [Resource Definitions — Flavor](../../architecture/resource-definitions.md#flavor)
- [Resource Definitions — Storage Class](../../architecture/resource-definitions.md#storage-class)
- [Resource Definitions — Environment](../../architecture/resource-definitions.md#environment)
- [Resource Definitions — Volume](../../architecture/resource-definitions.md#volume)
- [Authorization — environment type](../../architecture/authz.md#environment)
- [MCP](../../architecture/mcp.md)
- [Runners — Runner Catalog](../../architecture/runners.md#runner-catalog)
- [Runners — Runner Selection](../../architecture/runners.md#runner-selection)
- [Agents Orchestrator](../../architecture/agents-orchestrator.md)
- [Egress Gateway](../egress-gateway/egress-gateway.md)
- [Secrets](../../architecture/secrets.md)
