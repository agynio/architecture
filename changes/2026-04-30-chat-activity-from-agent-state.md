# Chat Activity Status from Agent Activity, Not Container State

## Target

- [Runners — Workload Resource](../architecture/runners.md#workload-resource)
- [Runners — Workload State](../architecture/runners.md#workload-state)
- [Runners — Workload State Management](../architecture/runners.md#workload-state-management)
- [Runners — Agent Activity Sweep](../architecture/runners.md#agent-activity-sweep)
- [Chat — Activity Status](../architecture/chat.md#activity-status)

## Delta

The product spec defines the chat activity indicator's `Running` state as "An agent participant has an active workload that is currently processing the conversation." The current derivation in [Chat — Activity Status](../architecture/chat.md#activity-status) maps `Running` directly to `workload.status='running'`, but `workload.status='running'` only means the container is up and healthy. Between turns — after the agent finishes a response but before the orchestrator stops the workload (default `idle_timeout=5m`) — the workload sits in `status='running'` with the agent process idle, and the chat list shows `Running` for up to five minutes after the conversation went quiet.

The signal that distinguishes "container up" from "agent producing output" already exists: `agynd` calls `TouchWorkload` every `10s` while the agent CLI is producing output and stops while it waits, and `last_activity_at` reflects this. But Chat does not consult it, and there is no notification path for the `processing → idle` transition (`TouchWorkload` does not emit `workload.updated`).

Specifically:

- `Workload` has no field that summarizes "is the agent currently producing output". Chat would have to poll `last_activity_at` against a threshold itself, with no real-time signal for the `processing → idle` transition.
- `Chat.GetChats`'s `activity_status` derivation maps every `workload.status='running'` row to `Running`, including the long idle tail before the orchestrator stops the workload. The chat list misrepresents idle conversations as actively processing.
- The `running` workload status enum value is documented as a lifecycle state with no caveat that it does not imply agent activity. Other consumers (the Console workload list) inherit the same ambiguity.
- `TouchWorkload` does not emit a notification, so even if Chat derived activity from `last_activity_at`, the chat-app would have no event-driven path to refresh on `idle → processing` transitions.

## Acceptance Signal

- `Workload` carries an `agent_state` field with values `idle` and `processing`. It is initialized to `processing` on `CreateWorkload` (workloads are only created in response to unacked messages, so the agent is expected to begin producing output immediately) and is the authoritative answer to "is the agent currently producing output". It is orthogonal to `status`, which continues to track container lifecycle.
- The `Workload.status` enum documentation explicitly states that `running` reflects container lifecycle and does not imply agent activity, and cross-references `agent_state` for the activity signal.
- `TouchWorkload` is the sole writer of the `idle → processing` transition: when called and `agent_state='idle'`, it atomically updates `agent_state='processing'`, refreshes `last_activity_at`, and emits a `workload.updated` event on the workload's organization topic. When `agent_state` is already `processing`, only `last_activity_at` is written and no event is emitted (steady-state path stays cheap).
- The Runners service runs an Agent Activity Sweep at `5s` intervals — the sole writer of the `processing → idle` transition. It selects workloads where `status='running' AND agent_state='processing' AND last_activity_at < now() - KEEPALIVE_GRACE AND removed_at IS NULL` and updates each to `agent_state='idle'`, emitting `workload.updated`. The sweep does not interact with runners — it is a DB scan plus notification emit — so the Runners service classification as a passive store is preserved.
- `KEEPALIVE_GRACE` defaults to `25s` (above the agynd keepalive interval of `10s` with tolerance for one missed beat). Maximum delay between the agent stopping and the chat indicator transitioning to `finished` is `KEEPALIVE_GRACE + sweep interval` — `~30s` by default, vs. up to `5m` today.
- The Agent Activity Sweep is independent of the orchestrator's [Idle Timeout](../architecture/runners.md#idle-timeout) enforcement. The sweep flips the activity bit after seconds (chat-indicator scope); the orchestrator stops the workload entirely after `idle_timeout` (default `5m`, lifecycle scope). Both consume the same `last_activity_at` signal but on different timescales.
- On the `starting → running` status transition, `UpdateWorkload` resets `last_activity_at = now()` so the sweep gives `agynd` a fresh keepalive window to make its first `TouchWorkload`. Without this reset, slow image pulls could push `last_activity_at` (initialized to `created_at`) past `KEEPALIVE_GRACE` before the agent ever boots, and the sweep would flip a healthy newly-running workload to `idle` immediately.
- `Chat.GetChats`'s `activity_status` derivation splits the `status='running'` row into two: `status='running' AND agent_state='processing'` contributes `running`; `status='running' AND agent_state='idle'` contributes `finished`. All other rows (`starting`, `stopping`, `failed`, `stopped`, no workload) are unchanged. The `running > pending > finished` aggregation across non-passive agent participants is unchanged.
- The chat-app continues to subscribe to `workload:{id}` rooms via `active_workload_ids` from `GetChats`. No new subscription model — `agent_state` transitions ride on the existing `workload.updated` event and trigger the same `GetChats` refetch path described in [Chat — Real-time updates](../architecture/chat.md#real-time-updates).
- `agynd` behavior and documentation are unchanged. It already calls `TouchWorkload` only while the agent is processing; all new behavior is server-side (the transition is computed on receipt and the sweep flips it back). `agynd` does not need to know about `agent_state` or about Chat as a consumer.

## Notes

The product spec wording for `Running` ("currently processing the conversation") is already correct under this change — only the implementation needs to match the wording. No product-spec change is required.

Workload `status` enum values are unchanged. Renaming `running` was considered to remove the chat-vs-container ambiguity but rejected: the term is industry-standard for container runtimes and the new `agent_state` field carries the additional semantic without forcing a wide rename across the Console, Runners service, runner protocol, Terraform provider, and the orchestrator's reconciliation code paths.

Adding a small background loop to the Runners service is a deliberate deviation from its prior "passive store" framing. It is justified because the `processing → idle` transition is an internal denormalization of `last_activity_at` (which Runners already owns) and avoids forcing every reader — Chat, Console, future consumers — to implement their own threshold logic and miss the notification path. The sweep does not talk to runners directly, so the broader passive-store classification still holds.

The default values (`KEEPALIVE_GRACE=25s`, sweep interval `5s`) can be tuned without API changes — they are server-side knobs.
