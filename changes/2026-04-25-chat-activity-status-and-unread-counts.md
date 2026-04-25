# Chat Activity Status and Per-Chat Unread Counts

## Target

- [Chat product spec — Activity Status](../product/chat/chat.md#activity-status)
- [Chat product spec — Conversation List](../product/chat/chat.md#conversation-list)
- [Chat product spec — Conversation Area](../product/chat/chat.md#conversation-area)
- [Chat service — Activity Status](../architecture/chat.md#activity-status)
- [Chat service — Active Workload IDs](../architecture/chat.md#active-workload-ids)
- [Chat service — Unread Counts](../architecture/chat.md#unread-counts)
- [Chat service — Marking Messages as Read](../architecture/chat.md#marking-messages-as-read)
- [Chat service — Partial Failure Handling](../architecture/chat.md#partial-failure-handling)
- [Threads service — `GetUnackedMessageCounts`](../architecture/threads.md#interface)
- [Authorization — Threads Service](../architecture/authz.md#threads-service)

## Delta

### Activity status

The product spec defines an `activity_status` of `running`, `pending`, or `finished` for chats with non-passive agent participants, derived per the [Chat — Activity Status](../architecture/chat.md#activity-status) rules from the most recent workload on each `(thread, agent)` pair. Current state:

- `Chat.GetChats` returns no `activity_status` field on chat entries.
- The chat-app renders the indicator with a hard-coded `pending` value for every chat — so every conversation appears as Pending regardless of the underlying workload.
- The Chat service does not call `Runners.ListWorkloadsByThread` and has no dependency on Runners. The dependency arrow `Chat → Runners` does not exist yet.
- The chat-app does not subscribe to `workload.updated` events for visible chats, so even with a correct initial value the indicator would not transition.
- The chat-app does not optimistically render `pending` on the user's outgoing send, so a brief `finished` flash would otherwise be visible during the gap between `SendMessage` and the orchestrator creating a `starting` workload.

### Per-chat unread counts

The product spec defines an unread message count badge on each conversation list entry, hidden when zero, and a user story for spotting conversations with new activity at a glance. Current state:

- `Chat.GetChats` returns no `unread_count` field on chat entries.
- The chat-app's conversation list does not render an unread badge — there is no element in `ChatListItem` for the count.
- `Chat.GetMessages` already exposes an unread count for the open chat; that path is unaffected.
- `Threads` exposes `GetUnackedMessages` (full message bodies) but no count-only RPC. Aggregating per-thread counts on the Chat service via `GetUnackedMessages` would force pagination of message bodies just to count them and is bounded by the page size cap. A new self-only `Threads.GetUnackedMessageCounts(participant_id) → map<thread_id, count>` is added to serve the badge directly. It is backed by the existing `(participant_id, acked_at)` index — no new index, no new auth model entry beyond the self-only check that mirrors `GetUnackedMessages`.

### Workload room subscriptions for the chat-app

The chat-app needs to subscribe to `workload:{id}` rooms to receive `workload.updated` events for chats that are currently visible. Current state:

- The chat-app has no source of workload IDs — `GetChats` returns none, and the chat-app does not call `Runners` directly (and should not, per the chat-app boundary).
- `Chat.GetChats` will include `active_workload_ids: list<string>` per chat — the IDs of workloads currently in `starting`/`running`/`stopping`. These are a free side product of the `ListWorkloadsByThread` calls already issued to derive `activity_status`.
- The `workload:{id}` notification-room auth is relaxed back to `member` on the workload's organization. [Server-side activity lists, 2026-04-24](2026-04-24-activity-server-side-lists.md) tightened it to `can_view_workloads` (owner-only) along with `ListWorkloads` / `GetWorkload` / `StreamWorkloadLogs`. The room only carries workload status events — actual workload data and logs still require those heavy RPCs, which keep the `can_view_workloads` gate. Reverting the room to `member` lets thread participants subscribe without exposing anything that wasn't already accessible to them. This also realigns [Authorization — Notifications Service](../architecture/authz.md#notifications-service) with [Notifications — Room Naming](../architecture/notifications.md#room-naming-convention) (which still says `member`).

### Partial failure handling for `GetChats`

`GetChats` newly fans out to Threads (counts), Runners (workloads), and Identity/Users/Agents (participant profiles). Current state:

- The Chat service has no defined behavior for partial failures of these calls — today it does not call most of them.
- The chosen contract: Threads is a hard dependency (its failures propagate as a `GetChats` error); Runners and Identity/Users/Agents are soft (failures degrade affected rows to `finished` / drop unresolved profile fields, log with enough detail to diagnose, and let the next refresh recover). See [Chat — Partial Failure Handling](../architecture/chat.md#partial-failure-handling) for the full table. Without this contract, a single Runners hiccup would render the whole conversation list blank for every user on the platform.

### Mark as read on conversation open

The product spec defines that opening a conversation marks every message in it as read, and that messages received while the conversation is open are marked as read on arrival. Current state:

- The chat-app does not call `Chat.MarkAsRead` when the conversation view mounts.
- The chat-app does not call `Chat.MarkAsRead` on `message.created` events for the currently-open conversation.
- As a consequence, even after `unread_count` lands on `GetChats`, badges would not clear when the user reads a conversation.

### Inconsistent activity-status terminology

The product spec previously described the activity indicator twice with different wordings — `(running, pending, finished)` in the conversation list section and `(working → waiting → idle)` in the real-time updates section. The spec is now unified on `running / pending / finished`. No code change is required for this — it is a documentation fix, listed here only to call out that any UI strings or comments still referencing `working`, `waiting`, or `idle` should be migrated to the canonical names.

## Acceptance Signal

- `Threads.GetUnackedMessageCounts(participant_id) → map<thread_id, count>` exists and is self-only (`participant_id == caller.identity_id`). It is documented in the Threads interface and in the Threads-service authorization table.
- `Chat.GetChats` response items include `activity_status` (`running` | `pending` | `finished` | `null`), `unread_count` (non-negative integer), and `active_workload_ids` (list of workload IDs currently in `starting`/`running`/`stopping`; empty list when none) fields.
- `activity_status` is `null` for chats with no non-passive agent participant and for `degraded` threads; otherwise it is computed per the rules in the [Activity Status](../architecture/chat.md#activity-status) section, with the `running` > `pending` > `finished` aggregation across agent participants.
- For a single `GetChats` page, the Chat service issues one `Threads.GetUnackedMessageCounts(participant_id=caller)` call for caller-side unread counts and one `Runners.ListWorkloadsByThread` call per `(thread, non-passive agent)` pair on the page in parallel. No serial per-chat round trip; the `(thread, agent)` results are joined in memory and the workload IDs that drive `activity_status` are also exposed as `active_workload_ids` (no extra round trips).
- `GetChats` partial-failure handling matches [Chat — Partial Failure Handling](../architecture/chat.md#partial-failure-handling): Threads failures propagate; Runners failures degrade affected `(thread, agent)` pairs to `finished` with empty workload IDs and a structured log; Identity/Users/Agents failures drop the unresolved participant's display fields with a structured log. A single backend hiccup does not blank the whole page.
- The chat-app's conversation list renders the activity indicator from the response field and an unread badge that is hidden when `unread_count == 0`.
- The chat-app subscribes to `workload:{id}` rooms for every ID in `active_workload_ids` of currently-visible chats, and unsubscribes when an ID drops off the list (chat scrolls out of view, refresh removes the workload).
- [Authorization — Notifications Service](../architecture/authz.md#notifications-service) gates `workload:{id}` rooms on `member` (not `can_view_workloads`). A regular org member who is a thread participant can subscribe; `ListWorkloads` / `GetWorkload` / `StreamWorkloadLogs` continue to require `can_view_workloads` (no change to those).
- Opening a conversation triggers exactly one `Chat.MarkAsRead` call for its thread.
- While a conversation is open, every `message.created` event on `thread_participant:{caller}` for that thread triggers another `Chat.MarkAsRead` call.
- When a conversation's underlying workload transitions, the chat-app refreshes that chat's `activity_status` from `GetChats` so the indicator updates without page reload — driven by `workload.updated` events on `workload:{id}` rooms and `message.created` events on `thread_participant:{caller}`.
- The chat-app renders `pending` optimistically on the sender's own outgoing message until the next `GetChats` refresh confirms the server-derived status — eliminating the `finished → pending` flash on send.
- No UI surface (string, comment, or component name) references `working`, `waiting`, or `idle` for the chat activity indicator.

## Notes

The dependency on Runners is new for Chat. The OpenFGA permissions already cover the `ListWorkloadsByThread` call (`member` on `organization:<workload.org_id>`, which thread participants satisfy). No new authorization tuple writes are required.

The `workload:{id}` room auth relaxation rolls back one of the changes in [Server-side activity lists, 2026-04-24](2026-04-24-activity-server-side-lists.md). The other tightenings from that change — `ListWorkloads`, `GetWorkload`, `StreamWorkloadLogs`, `ListVolumes`, `GetVolume`, `ListOrganizationThreads`, the Console acceptance signals, and the `can_view_workloads` / `can_view_volumes` / `can_view_threads` relations themselves — are unchanged.
