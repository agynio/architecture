# Chat: Editable Summary and UI Simplification

## Target

[Chat Product Spec](../product/chat/chat.md)

## Delta

### Editable conversation summary

The product spec defines the conversation summary as editable by any participant, displayed in the conversation list and detail header. Current state:

- The Chat service data model has no summary field. The `Chat` store type contains only `ThreadID`, `OrganizationID`, and `CreatedAt`.
- The Chat API has no RPC to update a conversation summary.
- The chat-app UI displays participant names as the list entry title. There is no summary field in the `Chat` API response type or the `ChatListItem` UI model.

### Open / Resolved filter

The product spec defines filtering conversations by status (Open, Resolved, All) and toggling status via a dropdown in the detail header. Current state:

- The segmented control and status dropdown exist in the UI but are not wired to the backend. The Chat service API (`GetChats`) has no status filter parameter. The `Chat` data model has no status field — there is no `UpdateChat` RPC to change status.

### Removed: run count and container count badges

The product spec no longer includes run count or container count badges in the conversation detail header. Current state:

- The chat-app UI renders run count and container count badges in `ChatDetailHeader`.

### Removed: Container Terminal

The product spec no longer includes the Container Terminal feature. Current state:

- The chat-app UI implements `ContainerTerminalDialog` with container listing, terminal modal, and container switching.

### Removed: collapsible run info panel

The product spec no longer includes the right-side run info column (status, duration, tokens, cost, run timeline link). Current state:

- The `Chat` component renders a collapsible right column with `RunInfo` for each run. The `ChatDetailHeader` includes a toggle button (`PanelRight`/`PanelRightClose`) to collapse/expand it.

### Removed: inline runs in conversation area

The product spec no longer shows runs inline between messages. Current state:

- The chat-app groups all messages into a single synthetic "run" for display purposes and renders run status/duration from `useChatRuns`.

### Real-time message delivery

The product spec states that new messages appear in the conversation without page refresh. Current state:

- The `GraphSocket` class handles `message_created` events but real-time sockets are disabled in production (`socketsEnabled = !import.meta.env.PROD`).

## Acceptance Signal

- Conversation summary is stored, auto-generated, editable, and displayed in the list and header.
- Open/Resolved filter and status toggle function end-to-end.
- Run count badge, container count badge, Container Terminal, run info panel, and inline runs are removed from the chat-app.
- Real-time WebSocket delivery is enabled in production; new messages appear without page refresh.
