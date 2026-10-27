# Workload Hibernation on a Pool of MicroVM Slots

## Target

- [vm-runner](../architecture/vm-runner.md) — new document
- [Runner — Workload Lifecycle](../architecture/runner.md#workload-lifecycle)
- [Runners — Workload Resource](../architecture/runners.md#workload-resource), [Workload State Management](../architecture/runners.md#workload-state-management), [Idle Timeout](../architecture/runners.md#idle-timeout), [Agent Activity Sweep](../architecture/runners.md#agent-activity-sweep)
- [Agents Orchestrator — Hibernation](../architecture/agents-orchestrator.md#hibernation), [Start Decision](../architecture/agents-orchestrator.md#start-decision), [Hibernate and Wake Flows](../architecture/agents-orchestrator.md#hibernate-and-wake-flows), [Metering](../architecture/agents-orchestrator.md#metering), [Workload Reconciliation](../architecture/agents-orchestrator.md#workload-reconciliation)
- [agynd — Turn Boundary](../architecture/agynd-cli.md#7-turn-boundary), [Wake](../architecture/agynd-cli.md#8-wake)
- [Resource Definitions — Agent](../architecture/resource-definitions.md#agent), [Sandbox](../architecture/resource-definitions.md#sandbox)
- [OpenZiti — Agent Identity Lifecycle](../architecture/openziti.md#agent-identity-lifecycle)
- [Chat — Activity Status](../architecture/chat.md#activity-status)
- [Sandboxes — Lifecycle](../product/sandboxes/sandboxes.md#lifecycle)

## Delta

**An idle workload costs its flavor until something stops it, and a stopped one costs a cold start to come back.** The only runner is the [k8s-runner](../architecture/k8s-runner.md): one Pod per workload, alive at full allocation from the first inbox item until the class's `idle_timeout` (default five minutes) stops it, then gone — processes, shells, scratch space and all — until the next item pays for a Pod again. Pod startup is measured at around eleven seconds; `runtimeClassName: kata-fc` adds a VM boundary but changes neither number, because the VM belongs to Kata's shim and nothing in the platform can pause it. An agent that answers a message every few hours therefore either idles at full cost between turns or cold-starts for every one of them, and nothing in between exists: no Runner RPC, no workload status, no record field, no decision in the Orchestrator knows a workload that is neither running nor stopped.

Desired state:

| | |
|---|---|
| **vm-runner** | A second Runner implementation: one Firecracker microVM per workload, hosted in a pool of prewarmed, unprivileged slot Pods; the stock Kata guest agent inside, driven by the runner's own slot supervisor without the Kata shim; disks served by the supervisor as base plus delta; snapshots, templates and volume deltas in an S3-compatible store; a guest kernel with `CONFIG_VMGENID`. New workloads start from a per-environment template; idle ones hibernate; hibernated ones wake in any slot |
| **Runner API** | `HibernateWorkload` and `WakeWorkload`, and `hibernating` / `hibernated` / `waking` in the reported status vocabulary. `StopWorkload` on a hibernated workload discards its state. Runners that cannot hibernate return `Unimplemented` and advertise nothing |
| **Runners service** | The three statuses on the workload record, with `hibernated_at`, `woken_at` and `snapshot_size_bytes`; the `waking → running` transition carrying its resets; `ReportWorkloadIdle`, the turn-boundary report that flips `agent_state` without waiting for the sweep; `wake_failed` and `snapshot_lost` failure reasons; `pending_sample` covering hibernation tails and snapshot storage; `hibernate` as an advertised capability the Orchestrator reads |
| **Agents Orchestrator** | The hibernation decision — turn ended, inbox empty, runner capable, `hibernate_after` elapsed — and the wake path in the Start Decision; hibernation TTL enforcement; identity kept across sleep and pull credential revoked and re-minted; reconciliation rows for the three statuses; compute metering that ends at `hibernated_at` and resumes at the wake, with snapshot storage billed in between; the sandbox layout snapshot taken before hibernation too |
| **agynd** | `ReportWorkloadIdle` after each turn's ack; `SIGCONT` as the wake cue to resubscribe and refetch the inbox |
| **Agent class** | `hibernate` (default on), `hibernate_after` (default `0s`) and `hibernation_ttl` (default seven days) beside `idle_timeout`, which becomes the stop of last resort on a capable runner |
| **Sandboxes** | `hibernated` as a sandbox status; idle timeout hibernates where the runner can; `EnsureSandboxRunning` wakes |
| **Terminal Proxy** | TTY sessions touch agent workloads as they touch sandboxes, so a person attached to an agent is never frozen mid-session |
| **Chat** | `activity_status` understands the three statuses |

### Boundaries

Hibernation is something a runner advertises and the Orchestrator decides. The k8s-runner advertises nothing and keeps its behaviour to the letter; an environment on it sees no change. The Orchestrator decides because only it can see that an inbox is empty, which is the one fact that makes a pause safe; the runner persists and restores and never reasons about work.

The workload spec is unchanged. The vm-runner consumes what the Orchestrator assembles today — containers, sidecars, init containers, volumes, inline files, pull credential, flavor name — and realizes it inside a guest. Nothing about what a workload contains, how MCP sidecars are reached, how the Ziti sidecar enrolls or how egress is policed is redefined.

### What does not change

The idle timeout, the activity sweep, the Start Decision's retry policy, volume reconciliation and TTL, identity orphan reconciliation and the metering sampler all keep their shape; each gains the hibernated case rather than a parallel mechanism. Pull credentials stay per workload and short-lived. Exposed ports and egress rules hang off the identity and survive sleep with it.

## Acceptance Signal

On a vm-runner, an agent instance that has just answered a message shows a workload that goes `running → hibernating → hibernated` within seconds of the turn's ack, emits no `FLAVOR_SECONDS` records while asleep and emits `resource=snapshot` storage instead; the next message produces a reply from the **same** workload record — `hibernated → waking → running`, no new `starting` record — and the time from the message's `accepted_at` to the agent's first LLM call is a small fraction of a cold start on the same hardware, on the order of a second. A sandbox left unattended past its idle timeout shows `hibernated`, and `connect` returns to the same shells with their processes still running.

A hibernated workload stopped by `hibernation_ttl` leaves its volumes in place and the next message cold-starts on them, as any stop does today. On a k8s-runner, an agent's workload list is indistinguishable from before this change.

## Notes

Validated on 2026-10-27 in a nested-KVM k3s VM (Firecracker 1.17, Kata 3.32 guest agent driven without the shim, unprivileged k3s slot Pods with a device plugin): pause 5–9 ms, `snapshot/load` 8–19 ms, differential snapshot 437 ms for 64 MiB dirtied, hibernate in one Pod and wake in another with processes and egress intact, two clones of one template with distinct injected identities. Full snapshot creation ran at 2–3 s per GiB and first-touch population after a wake at about 1 ms per fault under nested virtualization; both are expected to be one to two orders of magnitude cheaper on production hardware and must be measured there before the wake budget in the acceptance signal is committed to.

Decisions taken: Firecracker rather than Cloud Hypervisor, because production already runs it and its full and differential snapshots are generally available; the platform's own slot supervisor rather than adopting [agent-substrate](https://github.com/agent-substrate/substrate), which is early, Cloud Hypervisor-based and carries a control plane of its own; slots as Pods rather than a privileged node daemon, so that the scheduler accounts for every guest.

Two install surfaces follow the runner rather than precede it: the workload NetworkPolicy template needs egress to the image proxy and the snapshot store, and the local bundle's image bake fills the node's container store, which this runner does not read — the bundle either extends the bake to the runner's cache or accepts a first-use pull through the proxy. To verify on production nodes before building the vm-runner: that the shipped Firecracker supports snapshot and restore of vhost-user block devices, failing which the supervisor serves disks as reflinked files and tracks deltas itself; userfault-driven page population from the object store; KVM and the `vhost_vsock` module on the Magnum node image; the object store the runner is given. The Kata guest kernel ships without `CONFIG_VMGENID`, which is why the runner builds its own — a prerequisite for templates, not an option.

Depends on [runner-reported workload state](2026-08-11-runner-reported-workload-state.md): a hibernation is complete when the runner says so, and a wake is a transition the chat indicator follows within a second, not a reconcile interval. Sequenced in four steps after it, each leaving the platform working: the Runner API and the Runners and Orchestrator surfaces first, with the k8s-runner unaffected and nothing yet advertising `hibernate`; then the vm-runner with cold boot, hibernate and wake; then templates; then differential snapshots with consolidation. Server-side enrollment of the workload's OpenZiti identity — handing the guest an enrolled certificate instead of a JWT — would take the remaining second out of a template start and is a change of its own.
