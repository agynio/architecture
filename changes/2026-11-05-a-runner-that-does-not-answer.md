# A Runner That Does Not Answer

## Target

- [Agents Orchestrator — A runner that does not answer](../architecture/agents-orchestrator.md#a-runner-that-does-not-answer)
- [Agents Orchestrator — Hibernation](../architecture/agents-orchestrator.md#hibernation)
- [Agents Orchestrator — Volume Reconciliation](../architecture/agents-orchestrator.md#volume-reconciliation)

## Delta

The workload reconciliation table only described a runner that answered. The Orchestrator filled the gap by treating a runner with no OpenZiti terminator as having lost everything it held: hibernated workloads went to `snapshot_lost`, running ones to `runtime_lost`, and their identities were deleted. That came in with the implementation of [Degraded Thread Status](2026-04-20-degraded-thread-status.md), for a runner reprovisioned under a new ID, whose old service never gets a terminator again.

A runner that is only restarting has no terminator either, until it has re-enrolled, and still holds all of it. One reconciliation tick inside that gap wrote it off. Hibernation makes that expensive: it exists to keep an idle agent's state, and the state was thrown away while the runner still had it.

Desired state:

- A failed `Runner.ListWorkloads` concludes nothing. Records stay as they are and the next tick asks again.
- A runner is taken to be gone only once it has stayed without a terminator for `RUNNER_LOST_AFTER` (`30m`), without once answering. Its workloads are then reconciled as missing.
- A wake the runner did not answer leaves the workload `waking`, with nothing discarded, and reconciliation retries. A stop the runner did not answer stays `stopping` and is retried the same way.
- Volume reconciliation reads an unanswered runner the same way.

A runner no longer `enrolled` is unchanged: its status already says it is gone.

## Acceptance Signal

- A hibernated workload stays `hibernated` through a runner outage longer than a reconciliation tick, and wakes from its snapshot afterwards.
- A message that arrives during the outage is answered by that same workload once the runner is back, with no `wake_failed` and no cold start.
- A platform upgrade leaves hibernated workloads `hibernated`.
- A runner that stays without a terminator past `RUNNER_LOST_AFTER` has its workloads marked lost, as before.

## Notes

Observed on the local bundle VM on 2026-11-03 and 2026-11-04: every platform upgrade failed every hibernated workload with `snapshot_lost`, in one case in the 9 seconds between the restarted Orchestrator's first tick and the restarted runner regaining its terminator. Taking the runner down for 90 seconds did the same with no upgrade involved.

`RUNNER_LOST_AFTER` trades how long a truly gone runner's threads wait for how long an outage survives. A gone runner is rare and a restart is routine, so it is set well past the longest restart seen, about ten minutes with the overlay restarting too.
