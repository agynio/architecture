# Degraded Thread Status

## Target

- [Threads](../architecture/threads.md)
- [Agents Orchestrator — Runner Selection](../architecture/agents-orchestrator.md#runner-selection)
- [Agents Orchestrator — Volume Reconciliation](../architecture/agents-orchestrator.md#volume-reconciliation)
- [Chat — Degraded Threads](../architecture/chat.md#degraded-threads)
- [Telegram Connector — Thread Mapping](../architecture/apps/telegram-connector.md#thread-mapping)

## Delta

### Threads service

- Added `degraded` to the `Thread.status` enum.
- Added `DegradeThread(thread_id, reason)` internal method. Callable only by the Agents Orchestrator (Istio-internal).
- `SendMessage` returns an error for both `archived` and `degraded` threads. Read operations remain available.
- Added **Thread Status** section documenting the semantics of all three statuses.

### Agents Orchestrator — runner selection

- Before starting a workload, the Orchestrator checks `Runners.ListVolumesByThread(thread_id)` for existing volumes.
- If volumes exist, their `runner_id` is used as the predetermined runner. The Orchestrator validates the runner is still enrolled via `Runners.GetRunner`.
- If the runner is no longer registered or not enrolled, the Orchestrator calls `Threads.DegradeThread(thread_id, reason="runner_deprovisioned")` and aborts the start sequence instead of attempting to contact the missing runner.

### Agents Orchestrator — volume reconciliation

- When an active volume's PVC is not found on the runner (`active` → `failed` transition), the Orchestrator calls `Threads.DegradeThread(thread_id, reason="volume_lost")`.

### Chat UI

- When `SendMessage` returns a degraded error, the UI displays an inline banner indicating the thread is unavailable due to an infrastructure failure.

### Telegram Connector

- When `SendMessage` returns a degraded error, the connector deletes the `(installation_id, telegram_chat_id)` mapping and creates a new thread as if no mapping existed. The triggering message is forwarded to the new thread.

## Acceptance Signal

- `Thread.status` has a `degraded` value.
- `Threads.DegradeThread` RPC exists and is callable internally.
- `Threads.SendMessage` returns an error for degraded threads.
- Orchestrator runner selection validates runner enrollment before use and degrades the thread if the runner is gone.
- Orchestrator volume reconciliation degrades the thread on volume loss.
- Chat UI renders a banner for degraded threads.
- Telegram Connector recreates the thread mapping on a degraded error.
