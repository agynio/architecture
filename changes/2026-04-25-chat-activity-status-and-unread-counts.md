# Chat Activity Status and Per-Chat Unread Counts

## Target

- [Chat product spec — Activity Status](../product/chat/chat.md#activity-status)
- [Chat product spec — Conversation List](../product/chat/chat.md#conversation-list)
- [Chat product spec — Conversation Area](../product/chat/chat.md#conversation-area)
- [Chat service — Activity Status](../architecture/chat.md#activity-status)
- [Chat service — Unread Counts](../architecture/chat.md#unread-counts)
- [Chat service — Marking Messages as Read](../architecture/chat.md#marking-messages-as-read)

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

### Mark as read on conversation open

The product spec defines that opening a conversation marks every message in it as read, and that messages received while the conversation is open are marked as read on arrival. Current state:

- The chat-app does not call `Chat.MarkAsRead` when the conversation view mounts.
- The chat-app does not call `Chat.MarkAsRead` on `message.created` events for the currently-open conversation.
- As a consequence, even after `unread_count` lands on `GetChats`, badges would not clear when the user reads a conversation.

### Inconsistent activity-status terminology

The product spec previously described the activity indicator twice with different wordings — `(running, pending, finished)` in the conversation list section and `(working → waiting → idle)` in the real-time updates section. The spec is now unified on `running / pending / finished`. No code change is required for this — it is a documentation fix, listed here only to call out that any UI strings or comments still referencing `working`, `waiting`, or `idle` should be migrated to the canonical names.

## Acceptance Signal

- `Chat.GetChats` response items include `activity_status` (`running` | `pending` | `finished` | `null`) and `unread_count` (non-negative integer) fields.
- `activity_status` is `null` for chats with no non-passive agent participant and for `degraded` threads; otherwise it is computed per the rules in the [Activity Status](../architecture/chat.md#activity-status) section, with the `running` > `pending` > `finished` aggregation across agent participants.
- For a single `GetChats` page, the Chat service issues one `Threads.GetUnackedMessages(participant_id=caller)` call for caller-side unread counts and one `Runners.ListWorkloadsByThread` call per `(thread, non-passive agent)` pair on the page in parallel. No serial per-chat round trip; the `(thread, agent)` results are joined in memory.
- The chat-app's conversation list renders the activity indicator from the response field and an unread badge that is hidden when `unread_count == 0`.
- Opening a conversation triggers exactly one `Chat.MarkAsRead` call for its thread.
- While a conversation is open, every `message.created` event on `thread_participant:{caller}` for that thread triggers another `Chat.MarkAsRead` call.
- When a conversation's underlying workload transitions, the chat-app refreshes that chat's `activity_status` from `GetChats` so the indicator updates without page reload — driven by `workload.updated` events on `workload:{id}` rooms and `message.created` events on `thread_participant:{caller}`.
- The chat-app renders `pending` optimistically on the sender's own outgoing message until the next `GetChats` refresh confirms the server-derived status — eliminating the `finished → pending` flash on send.
- No UI surface (string, comment, or component name) references `working`, `waiting`, or `idle` for the chat activity indicator.

## Notes

The dependency on Runners is new for Chat. The OpenFGA permissions already cover the call (`ListWorkloadsByThread` returns workloads for threads the caller participates in, and Chat passes the caller's identity through). No new authorization tuple writes are required.
