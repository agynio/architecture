# Deleting a Conversation

## Target

- [Chat — Deleting a Conversation](../product/chat/chat.md#deleting-a-conversation)
- [Chat — Header](../product/chat/chat.md#header)
- [Chat — Deleting a Chat](../architecture/chat.md#deleting-a-chat)
- [Chat — Interface](../architecture/chat.md#interface)
- [Chat — Authorization](../architecture/chat.md#authorization)
- [Authorization — Chat Service](../architecture/authz.md#chat-service)

## Delta

**A conversation, once created, is permanent.** Every mistaken `+`, every one-line question to an agent, every abandoned draft that got a message sent into it stays in the list forever. The only lever is Resolved, which is a lifecycle state, not a disposal — the conversation reappears the moment the filter is set to All. The list only grows.

The conversation actions menu gains **Delete conversation**, and Chat gains the `DeleteChat` RPC behind it.

The pieces to build:

- `DeleteChat` on Chat and on `ChatGateway`. It archives the thread, then marks the chat record deleted — in that order, so a half-failure leaves a visible-but-frozen chat rather than a hidden one agents can still write to.
- A deleted marker on the chat record, and `GetChats`, `UpdateChat`, and `GetMessages` resolving against it. Filtering deleted chats out client-side or by dropping them after the thread fetch would return short pages and break the cursor.
- The menu item, its confirmation dialog, and the navigation back to the empty state.

**Nothing in Threads changes.** `ArchiveThread` already exists, already means soft-delete, already refuses new messages on an archived thread, and already authorizes exactly the identities the product calls for — any participant, or an organization owner. No new RPC, no new relation, no new tuple. What is missing is a caller.

**Deletion is shared, not personal.** Chats are org-scoped objects whose status and summary are the same for everyone on them; deletion follows. Per-user hiding would be a different feature — a per-participant list state Chat does not have and this change does not add.

## Acceptance Signal

- A user opens a conversation, picks **Delete conversation** from the actions menu, confirms, and lands on the empty conversation state with the conversation gone from the list.
- Cancelling the dialog leaves the conversation and its messages untouched.
- The conversation is gone from every other participant's list on their next refresh, in every status filter including All.
- Navigating directly to a deleted conversation's URL resolves to the empty state rather than an error or an empty transcript.
- Sending into a deleted conversation is refused, from the app and from an agent.
- Deleting an already-deleted conversation succeeds and changes nothing, so a retried request never reports a failure for work that landed.
- A member of the organization who is not a participant cannot delete the conversation; an organization owner can.
- Deleting a conversation while its agent is working leaves the workload to idle out, and the reply the agent was writing never lands anywhere.
- The Console still finds the thread through `ListOrganizationThreads` with `status_in = [archived]`, with its messages intact.

## Notes

- **Deletion hides the conversation, it does not revoke reading it.** Chat's `GetMessages` refuses a deleted chat so a stale link knows to give up, but the messages stay in Threads and a participant holding the ID can still page them through `ThreadsGateway` — which is the path the app itself uses for a transcript. Redacting a conversation is a different feature.
- **No notification event.** Threads publishes nothing on archive, so a participant with the conversation open in another tab keeps rendering it until their next `GetChats` refresh. Sending from that stale view fails closed. A `thread.archived` event on `thread_participant:{id}` is the fix and is deliberately not specified here — the window is short and the failure is safe.
- **No undo.** The rows survive (this is a soft delete), so a restore path is possible later; nothing in the app offers one, and the confirmation dialog says so.
- **No bulk delete and no list-row action.** Deletion lives in the detail header only. Sweeping the list is a different interaction with a different confirmation story.
- **`UpdateChat` and the chat record's status and summary were never in the architecture doc.** They are now, alongside the new rows, so the interface and authorization tables describe the whole service.
