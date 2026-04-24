# Chat

## Purpose

Chat is the platform's communication interface. Users create conversations by selecting participants — other users, AI agents, or a mix of both — and exchanging messages. Every conversation lives in the same list regardless of who the participants are. When an agent is a participant, the conversation gains agent-specific capabilities such as reminders.

## User Stories

- As a user, I want to start a conversation with any combination of users and agents so all my communication happens in one place.
- As a user, I want to see all my conversations in a single list so I can find any discussion quickly.
- As a user, I want to filter conversations by status (open, resolved, all) so I can focus on active work.
- As a user, I want to send messages, markdown, and file attachments.
- As a user, I want to see images, video, and audio inline in messages so I can view media without leaving the conversation.
- As a user, I want to see charts and diagrams rendered inline in messages so I can understand data and structure at a glance.
- As a user, I want to see which messages have been read and which are unread.
- As a user, I want to see the number of unread messages on each conversation in the list so I can spot which ones have new activity at a glance.
- As a user, I want messages to be marked as read automatically when I open a conversation so I do not have to manage read state by hand.
- As a user, I want to see whether the agent is currently working on a conversation, waiting to start, or finished so I know whether to expect more output.
- As a user, I want to mark a conversation as resolved or reopen it.
- As a user, I want to delete messages I no longer need.
- As a user, I want to see reminders created by agents so I can track follow-ups.
- As a user, I want to cancel reminders if they are no longer needed.
- As a user, I want to edit the conversation summary so I can keep it meaningful as the discussion evolves.

## Layout

List-detail. Left panel: conversation list. Right panel: selected conversation.

Above the conversation list: the Agyn logo and an authenticated user button.

### User Menu

Clicking the user button opens a menu with:

- **Organization switcher** — lists all organizations the user has access to. The current organization is highlighted. Selecting a different organization switches context: the conversation list, selected conversation, and all org-scoped data reload for the newly selected organization.
- **Logout** — ends the session.

### Organization Context

All conversations are scoped to the selected organization. Switching organizations replaces the conversation list and clears the selected conversation.

### No Organizations

If the user has no available organizations, the entire chat interface (list and detail) is replaced by an onboarding screen. The screen informs the user that they need to either:

- **Get an invite** to an existing organization from another member, or
- **Create a new organization** — links to the configuration UI.

The user cannot access chat functionality until they belong to at least one organization.

## Creating a Conversation

Clicking "+" (new conversation) opens the composer:

1. A participant selector with search/autocomplete. The user picks one or more participants — users, agents, or both.
2. The user types the initial message.
3. Sending creates the conversation and navigates to it.

Drafts are local-only and discarded on cancel or navigation.

## Conversation Summary

Every conversation has a summary — a short text identifying the conversation. The summary is auto-generated from the initial message when the conversation is created. Any participant can edit the summary afterward.

The summary is the primary label shown in the conversation list and in the conversation detail header.

## Conversation List

All conversations, filtered by status: **Open** (default), **Resolved**, or **All**. Ordered by creation time (newest first). Each entry shows:

- Summary.
- Participant name(s).
- Creation timestamp.
- [Activity status indicator](#activity-status) — for conversations with agent participants.
- Unread message count badge — the number of messages the current user has not yet read in the conversation. Hidden when the count is zero.

Infinite scroll loads older conversations.

## Conversation Status

User-controlled lifecycle state — **Open** or **Resolved**. Toggled via dropdown in the detail header. Optimistic — the UI updates immediately and rolls back on failure.

## Activity Status

System-derived indicator that reflects whether agent participants are currently processing the conversation. Distinct from [Conversation Status](#conversation-status) (which is user-controlled). Shown only on conversations that have at least one non-passive agent participant; for user-only conversations, no indicator is shown.

| State | Meaning |
|-------|---------|
| **Running** | An agent participant has an active workload that is currently processing the conversation. |
| **Pending** | An agent participant has unread messages and is not yet actively processing — the workload is starting, retrying after a failure, or has not been started yet. |
| **Finished** | No agent participant has unread messages or an active workload. The most recent run has completed. |

When a conversation has multiple agent participants, the indicator reflects the most active state across them: **Running** wins over **Pending**, **Pending** wins over **Finished**.

Degraded conversations (see [Chat — Degraded Threads](../../architecture/chat.md#degraded-threads)) display the degraded banner instead of an activity status — the conversation cannot accept new messages and the indicator is hidden.

## Conversation Detail

### Header

- [Activity status indicator](#activity-status).
- Participant name(s) and role(s).
- Creation timestamp (relative).
- Summary (editable — click to edit inline, save on blur or Enter, cancel on Escape).
- Status toggle (Open / Resolved).
- Reminder count — with popover listing agent-created reminders, each cancellable.

### Conversation Area

Messages displayed chronologically. Each message shows sender, timestamp, and read/unread status. Messages can be deleted.

Opening a conversation marks every message in it as read for the current user. Messages received while the conversation is open are marked as read on arrival. The conversation's unread badge clears at the moment of open and stays clear until the user navigates away.

Each message has a **"View trace"** link. Clicking it opens the Run Timeline for that message. The tracing app resolves the run from the message ID and displays it.

Messages containing media (images, video, audio) — whether from external URLs or platform files — display the media inline. See [Inline Media](inline-media.md) for rendering behavior, security, and authentication details.

Messages containing `vega-lite` or `mermaid` fenced code blocks render those blocks as interactive charts or diagrams. See [Charts and Diagrams](charts-and-diagrams.md) for syntax, rendering behavior, and security.

When an agent is a participant, the conversation also shows:

- **Reminders** — scheduled follow-ups created by the agent, with content, scheduled time, and cancel action.

### Composer

A markdown editor at the bottom. Supports:

- Markdown formatting.
- Fenced code blocks, including `vega-lite` and `mermaid` blocks that render as [charts and diagrams](charts-and-diagrams.md) after sending.
- File attachments (drag-and-drop or file picker, 20 MB per file, with upload progress and retry on failure).
- Character limit indicator (warns when approaching or exceeding the limit).

### Message Drafts

The composer persists draft text per conversation to local storage. Drafts restore when returning to a conversation and clear after successful send.

## Real-Time Updates

Via WebSocket:

- New conversations appear in the list; summaries and statuses refresh.
- New messages appear in the conversation without page refresh.
- [Activity status](#activity-status) indicators transition between Running, Pending, and Finished as the underlying agent workload state changes.
- Unread counts on conversation list entries update as messages arrive and as the user reads them.
- Reminder counts refresh.
- On reconnection, all data re-fetches.

Sound notifications play on activity changes.

## Constraints

- File attachments: 20 MB per file.
- Message character limit enforced in the composer.
- Backend communication: ConnectRPC through the Gateway.

## Related architecture

- [Product to architecture map (Chat)](../../maps/product-to-architecture.md#chat)
- [Chat service](../../architecture/chat.md)
- [Threads](../../architecture/threads.md)
- [Notifications](../../architecture/notifications.md)
- [Media support](../../architecture/media.md)
- [Media proxy](../../architecture/media-proxy.md)
- [Reminders](../../architecture/apps/reminders.md)
