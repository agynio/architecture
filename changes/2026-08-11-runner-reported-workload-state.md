# Runner-Reported Workload State

## Target

- [Runners — Runner-Reported Workload State](../architecture/runners.md#runner-reported-workload-state)
- [Runners — Workload State](../architecture/runners.md#workload-state)
- [Runners — Workload State Management](../architecture/runners.md#workload-state-management)
- [k8s Runner — Workload State Reporting](../architecture/k8s-runner.md#workload-state-reporting)
- [Notifications — Cluster-wide rooms](../architecture/notifications.md#cluster-wide-rooms)
- [Agents Orchestrator — Reconciliation Loop](../architecture/agents-orchestrator.md#reconciliation-loop)

## Delta

**Nothing pushes runtime state to the platform.** The runner is the only component that can see a Pod, and it never says anything. The Orchestrator discovers that a workload started by dialing the runner and calling `InspectWorkload` on its reconcile interval, then writing the result with `UpdateWorkload`. So a workload is ready and serving for up to a full interval before anything records it, and the sandbox behind it stays `starting` for exactly that long — measured at up to 10s on an installation with `WORKLOAD_RECONCILE_INTERVAL=10s`, against ~11s of Pod startup. Roughly half the wait between starting a sandbox and its shell being usable is the platform not yet having looked.

`workload.status_changed` is published today and looks like the missing signal, but it is not: it fires from inside `UpdateWorkload`, whose only caller for the `running` transition is the Orchestrator itself. Subscribing to it would tell the Orchestrator what it had just concluded. The gap is a missing **publisher**, not a missing subscription.

Desired state:

| | |
|---|---|
| **`ReportWorkloadState`** | New runner-facing RPC on the Runners Gateway. A runner reports observed runtime state for workloads on itself — status, containers, `observed_at` — writing through the same path as `UpdateWorkload` so existing events fire unchanged |
| **Pod informer in k8s-runner** | A `SharedInformer` over Pods labelled `agyn.io/workload-id` in the runner's namespace, inside the existing process. One goroutine, one watch, no new Pod and no new RBAC — `watch` on Pods is already granted |
| **`workloads` room** | A third cluster-wide, platform-only Notifications room, alongside `agent_instances` and `sandboxes` |
| **Orchestrator wakes on it** | `workload.status_changed` wakes whichever loop owns the workload, instead of the transition being found by the next poll |

### Boundaries

A runner reports **observed** state only. It cannot mark a workload `stopped` that the platform intends to keep running — lifecycle stays with the Orchestrator. Reports carry `observed_at` and are dropped when older than the record they would overwrite, because retries and informer resyncs mean the same report arrives more than once and sometimes late.

### What does not change

The Orchestrator's reconciliation interval stays exactly as it is, and remains the backstop. Every reconcile loop already selects on both its wake channel and its ticker, so a lost report costs latency until the next tick and never correctness. A runner that is older than this RPC receives `Unimplemented` and keeps serving, exactly as catalog reporting already does — reporting is an optimization a runner offers, not a contract the platform requires.

## Acceptance Signal

Starting a stopped sandbox marks it `running` within roughly the round trip from the Pod becoming Ready, rather than on the next reconcile tick — observable as the gap between the Pod's `Ready` condition and the sandbox's `running` transition falling from up to one interval to under a second.

With the runner's reporting disabled or the runner stopped, the same transition still completes on the reconcile interval.

## Notes

Sequenced after the flat-room work already in `agents-orchestrator`, `agents` and `notifications`: `workloads` is the third room of that shape, and the Orchestrator's platform identity is what admits it to any of them.

The remaining latency after this change is Pod startup itself — seven sequential init containers, of which only ~3s is real work. That is a separate change: the three ziti script containers share one image and can be merged, three more only copy binaries into a shared volume and can run before the mesh waits rather than after, and the sandbox image is tagged `:latest`, which silently forces `imagePullPolicy: Always` and a registry round trip on every start.
