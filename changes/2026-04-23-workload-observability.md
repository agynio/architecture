# Workload Observability

## Target

- [Runners — Container](../architecture/runners.md#container)
- [Runners — Authorization](../architecture/runners.md#authorization)
- [Runners — Gateway Exposure](../architecture/runners.md#gateway-exposure)
- [Runner — Streaming](../architecture/runner.md#streaming)
- [Agents Orchestrator — Workload Reconciliation](../architecture/agents-orchestrator.md#workload-reconciliation)
- [Gateway — External API Surface](../architecture/gateway.md#external-api-surface)
- [Console — Gateway API Surface](../architecture/console.md#gateway-api-surface)
- [Console — Workloads](../product/console/console.md#workloads)

## Delta

Container-level observability is not usable for debugging. The Console shows every container as `Waiting (<N>)` regardless of actual runtime state, and container logs cannot be viewed from the Console. Wrong container images, failing MCP start commands, and crash loops are invisible to org admins.

Specifically:

- Container status is recorded at workload creation but never refreshed. Workload reconciliation updates only the workload-level status — the `containers` array keeps its initial placeholder values forever.
- `Container` has no `reason`, `message`, `exit_code`, `restart_count`, `started_at`, or `finished_at`. Even a refreshed array has nowhere to carry `ImagePullBackOff`, `CrashLoopBackOff`, `OOMKilled`, or exit codes.
- `Runner.StreamWorkloadLogs` is specified as follow-only and is not exposed through the Gateway. The Console has no path to container logs.
- Workload detail lists "container name, image, state, resource usage" — no reason, no message, no exit code, no log viewer.

## Acceptance Signal

- `Container` carries `reason`, `message`, `exit_code`, `restart_count`, `started_at`, and `finished_at` alongside `status`.
- On each workload reconciliation tick, the Orchestrator calls `Runner.InspectWorkload` for every workload present on the runner and persists the refreshed container list via `UpdateWorkload`. Per-container runtime state (including `ImagePullBackOff`, `CrashLoopBackOff`, `OOMKilled`, and exit codes) is visible within one reconciliation interval.
- `Runner.StreamWorkloadLogs` accepts `tail_lines`, `since_time`, and `follow` parameters and streams logs for a named container — snapshot and follow are the same RPC.
- `RunnersGateway.StreamWorkloadLogs` is exposed through the Gateway and authorized to members of the workload's organization. The Gateway resolves the hosting runner via `GetWorkload` and forwards the stream over OpenZiti.
- Console Workloads list summarizes container states per workload (e.g., `3 running, 1 waiting (ImagePullBackOff)`).
- Console Workload detail shows a per-container panel with state, reason, message, exit code, restart count, and started/finished timestamps, plus a log viewer that loads the last 1000 lines and then follows new output with a container selector. No tail-length or since-time controls.
- Log viewer shows a clear empty state when the container no longer exists on the runner.
