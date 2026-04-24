# Workload Start Retry Policy

## Target

- [Agents Orchestrator — Dependencies](../architecture/agents-orchestrator.md#dependencies)
- [Agents Orchestrator — Loop](../architecture/agents-orchestrator.md#loop)
- [Agents Orchestrator — Start Decision](../architecture/agents-orchestrator.md#start-decision)
- [Agents Orchestrator — Agent Start Flow](../architecture/agents-orchestrator.md#agent-start-flow)
- [Agents Orchestrator — Workload Reconciliation](../architecture/agents-orchestrator.md#workload-reconciliation)
- [Runners — Workload State](../architecture/runners.md#workload-state)
- [Runners — Workload Resource](../architecture/runners.md#workload-resource)
- [Agents Service — Notifications](../architecture/agents-service.md#notifications)
- [Threads — Interface](../architecture/threads.md#interface)
- [Threads — Thread Status](../architecture/threads.md#thread-status)

## Delta

A workload that failed to start stayed `failed` forever with no path to recovery. The main reconciliation loop had no retry logic, the workload reconciliation loop treated a CrashLoopBackOff/ImagePullBackOff pod as indistinguishable from a healthy one (pod was "present on runner"), and nothing propagated a fixed agent configuration to the Orchestrator. Threads blocked by the failure stayed blocked until an operator manually deleted the workload record — especially painful for single-threaded connectors like the Telegram app, where one bad config stopped all bidirectional messaging for a chat.

### Runners service

- Added `failure_reason` (enum: `start_failed`, `image_pull_failed`, `config_invalid`, `crashloop`, `runtime_lost`) and `failure_message` fields to the Workload resource. Populated by the Orchestrator when a workload transitions to `failed`.
- Extended `ListWorkloadsByThread` to accept optional `agent_id` and `status_in` filters and documented the `created_at DESC` ordering. Used by the Orchestrator's new Start Decision to inspect the most recent terminal workload for a `(thread_id, agent_id)` pair.

### Agents Orchestrator — Start Decision

- Added a new **Start Decision** subsection under Reconciliation that the "Start" branch of the main loop delegates to. The policy is derived entirely from workload history and the agent's `updated_at` — no retry state is persisted on any record.
- Consecutive failures are counted from workload records newer than `max(last_stopped.created_at, agent.updated_at)`. A successful idle stop or a post-failure config change resets the counter.
- Backoff schedule: `[10s, 30s, 1m, 5m, 15m]` (final entry repeats). Max attempts: `10`. With defaults, an unrecoverable configuration degrades the thread after roughly two hours.
- On `consecutive_failures >= MAX_ATTEMPTS`, the Orchestrator calls `Threads.DegradeThread(thread_id, reason="agent_start_failures_exhausted")` and stops retrying. Degraded threads are skipped by the main loop on subsequent ticks; consumer-side handling (e.g., the Telegram connector's thread rotation) takes over.

### Agents Orchestrator — configuration-driven fast retry

- Orchestrator subscribes to `agent.updated` on Notifications in addition to `message.created`. Updated Dependencies diagram and table accordingly.
- When the event fires, the main loop re-evaluates; the Start Decision's `agent.updated_at > latest.removed_at` check resets `consecutive_failures` to 1 immediately. A fixed configuration unblocks a backed-off thread within one tick.

### Agents Orchestrator — Workload Reconciliation

- Expanded the `starting` / `running` transition table with container-health-driven failure rows:
  - `ImagePullBackOff` / `ErrImagePull` past `START_GRACE_S` → `failed` (`image_pull_failed`).
  - `CreateContainerConfigError` / `CreateContainerError` / `InvalidImageName` past `START_GRACE_S` → `failed` (`config_invalid`).
  - Init container `exit_code ≠ 0` with `restart_count ≥ INIT_RETRY_THRESHOLD` → `failed` (`start_failed`).
  - Main container `reason=CrashLoopBackOff` with `restart_count ≥ CRASHLOOP_THRESHOLD` → `failed` (`crashloop`).
- Promotion `starting → running` is deferred until all init containers have completed and all main containers report `running`. A pod still in `ContainerCreating` / `PodInitializing` stays in `starting` with its container list refreshed.
- On `failed` transitions, the Orchestrator calls `Runner.StopWorkload` to clean up the pod and `ZitiManagement.DeleteIdentity` to release the OpenZiti identity, matching the cleanup sequence of the existing Agent Stop Flow.
- Added default thresholds: `START_GRACE_S=60`, `INIT_RETRY_THRESHOLD=3`, `CRASHLOOP_THRESHOLD=3`.

### Agents Orchestrator — Agent Start Flow

- `UpdateWorkload` after `StartWorkload` returns now sets only `instance_id`; it no longer sets `status=running`. The `starting → running` transition is owned by Workload Reconciliation based on observed container health.

### Agents service

- Added **Notifications** section documenting `agent.updated` emission on the `agent:{id}` room. The event fires when the agent resource or any of its sub-resources (MCP, Skill, Hook, ENV, InitScript, Volume, Volume Attachment, Image Pull Secret Attachment) is created, updated, or deleted.
- Documented transitive `updated_at` propagation: a write to any sub-resource bumps the owning agent's `updated_at` in the same transaction, so consumers comparing timestamps only need to read the agent record.

### Threads service

- `DegradeThread` description updated to list all recovery scenarios (volume lost, runner deprovisioned, agent start failures exhausted) and to note idempotency on already-degraded threads.
- `degraded` status description updated to name the machine-readable reason strings: `volume_lost`, `runner_deprovisioned`, `agent_start_failures_exhausted`.

## Acceptance Signal

- `Workload` carries `failure_reason` and `failure_message`, populated whenever the record transitions to `failed`.
- `ListWorkloadsByThread(thread_id, agent_id?, status_in?)` returns workloads ordered by `created_at DESC`.
- The Orchestrator subscribes to `agent.updated` and re-evaluates affected threads on each event.
- A workload stuck in `ImagePullBackOff` past `START_GRACE_S` transitions to `failed` with `failure_reason=image_pull_failed` within one reconciliation interval, and its pod is removed from the runner.
- A workload whose main container enters `CrashLoopBackOff` with `restart_count ≥ CRASHLOOP_THRESHOLD` transitions to `failed` with `failure_reason=crashloop` and its pod is removed.
- A thread with a failing agent is retried on the `BACKOFF_SCHEDULE` cadence up to `MAX_ATTEMPTS`, then degraded via `Threads.DegradeThread(..., reason="agent_start_failures_exhausted")`.
- When the agent configuration is corrected, `Agents` emits `agent.updated`; within one reconciliation tick the Orchestrator starts a fresh workload with `consecutive_failures = 1`, regardless of how long the prior backoff would have been.
- The Telegram connector rotates the thread on the `agent_start_failures_exhausted`-degraded thread via its existing `thread degraded` handling path — no code change required in the connector.
- The Agent Start Flow no longer sets `status=running`; workloads stay in `starting` until Workload Reconciliation promotes them based on container health.
