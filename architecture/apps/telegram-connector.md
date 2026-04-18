# Telegram Connector

## Overview

The Telegram Connector is a [platform app](../apps.md) that bridges Telegram and the platform. It connects a Telegram bot to a configured agent — Telegram users message the bot, the connector forwards messages to a platform thread, the agent responds, and the connector delivers the response back to Telegram. Media (photos, documents, audio, video, voice) is forwarded in both directions.

| Aspect | Detail |
|--------|--------|
| **Type** | [App](../apps.md) |
| **Identity** | `app` type in [Identity](../identity.md) |
| **Thread interaction** | Participant — creates threads, joins them, receives and sends messages |
| **Visibility** | `public` — any organization can install it |
| **Default slug** | `telegram` |
| **Deployment** | Independently deployed (not managed by the platform) |
| **Storage** | Own PostgreSQL database |
| **Connectivity** | [OpenZiti](../openziti.md) — dials Gateway |
| **Declared permissions** | `thread:create`, `participant:add` |

## Configuration

Each installation provides the following configuration:

| Key | Type | Description |
|-----|------|-------------|
| `bot_token` | string | Telegram Bot API token (from @BotFather) |
| `agent_id` | string (UUID) | Agent to add as participant when creating threads |

Multiple installations are supported — each with its own `bot_token` and `agent_id`. This allows one organization to run multiple bots (e.g., `telegram-support` and `telegram-sales`) from a single connector deployment.

## Thread Mapping

The connector maintains a persistent mapping of `(installation_id, telegram_chat_id) → thread_id`.

When a message arrives from a Telegram chat that has no mapping:
1. Create a new platform thread. The connector's app identity becomes a participant automatically as the creator.
2. Add the configured agent as a participant.
3. Store the mapping.

Subsequent messages from the same Telegram chat reuse the same thread. Thread lifecycle is indefinite — threads are never archived by the connector.

## Inbound Flow (Telegram → Platform)

The connector polls Telegram using long polling (`getUpdates`). One polling loop runs per installation.

### Installation Discovery

On startup the connector calls `ListInstallations` to enumerate all installations and starts one polling loop per result. A reconciliation loop runs every 60 seconds — it diffs the running loops against the current `ListInstallations` result, starts loops for new installations, and stops loops for uninstalled ones. No notification-driven hot-reload is needed; a 60-second lag for a newly installed bot is acceptable.

### Update Offset Persistence

The connector persists the last processed `update_id` per installation. Each `getUpdates` call passes `offset = last_update_id + 1`, which also instructs Telegram to discard all earlier updates. The offset is written to the database after each batch is successfully processed. On restart, the connector resumes from the stored offset, preventing duplicate inbound messages.

For each incoming Telegram update:

1. Look up or create the thread mapping for the chat.
2. Determine the message type and process accordingly (see [Media Handling](#media-handling)).
3. Call `SendMessage` on the platform Threads service via the Gateway.

### Message Types

| Telegram type | Platform message |
|---------------|-----------------|
| Text | `body: text`, no files |
| Photo + optional caption | `body: caption` (or empty), `files: [uploaded_id]` — largest available photo size |
| Document + optional caption | `body: caption` (or empty), `files: [uploaded_id]` |
| Audio + optional caption | `body: caption` (or empty), `files: [uploaded_id]` |
| Voice + optional caption | `body: caption` (or empty), `files: [uploaded_id]` |
| Video + optional caption | `body: caption` (or empty), `files: [uploaded_id]` |
| Sticker, contact, location, poll, and other types | `body: "[sticker]"` (or appropriate label), no files |

### Inbound Media Upload

For messages with a supported media type:

1. Call Telegram `getFile` to resolve the file path.
2. Download the file from `https://api.telegram.org/file/bot<token>/<file_path>`.
3. Upload to the platform Files service via the Gateway (`UploadFile`).
4. Include the returned file UUID in the `files` field of `SendMessage`.

## Outbound Flow (Platform → Telegram)

The connector follows the standard [Consumer Sync Protocol](../notifications.md#consumer-sync-protocol):

1. Subscribe to `message.created` notifications on the `thread_participant:{appId}` room.
2. On notification (or periodic poll), call `GetUnackedMessages`.
3. For each unacknowledged message:
   - Skip messages sent by the connector's own app identity (echo prevention).
   - Forward to the corresponding Telegram chat (see [Outbound Delivery](#outbound-delivery)).
4. Call `AckMessages` after successful delivery.

The `thread_id` on each message is used to look up the `telegram_chat_id` from the mapping table.

### Outbound Delivery

Text and files are delivered as separate Telegram messages.

**Body preprocessing:** before sending, the message `body` is scanned for inline markdown images (`![alt](url)`). Each match is extracted, removed from the body text, and queued as an additional photo to send. Two URL forms are handled:

- `agyn://file/<id>` — downloaded from the platform Files service and sent via `sendPhoto` (or `sendDocument` if the content type is not an image).
- `http(s)://` URL — passed directly to `sendPhoto` by URL (no download needed — Telegram fetches it).

**Delivery order:**

1. If the cleaned `body` is non-empty: send via `sendMessage`.
2. For each inline image extracted from the body: send via `sendPhoto` (or `sendDocument`).
3. For each file in `files`:
   - Download from the platform Files service via the Gateway (`GetFileContent`).
   - Detect the Telegram send method from the file's `content_type`:

| Content type | Telegram method |
|--------------|-----------------|
| `image/*` | `sendPhoto` |
| `audio/*` | `sendAudio` |
| `video/*` | `sendVideo` |
| `*` (voice, check filename) | `sendVoice` (if OGG/Opus) |
| Everything else | `sendDocument` |

## Delivery Failures

### Telegram Rate Limits

Telegram enforces a limit of 1 message per second per chat. When the API returns `429 Too Many Requests`, the response includes a `Retry-After` field indicating how many seconds to wait. The connector sleeps for that duration and retries the same send. If no `Retry-After` is present, exponential backoff is used (starting at 1 s, capped at 32 s).

### Bot Blocked by User

When a user blocks the bot, Telegram returns `403 Forbidden: bot was blocked by the user` on any outbound send attempt. The connector acknowledges the platform message immediately — retrying will never succeed and blocking the ack queue would stall delivery for all other chats. The chat mapping is marked as blocked. Outbound delivery to that chat is skipped until a new inbound message arrives from the user (which means they unblocked the bot), at which point the blocked state is cleared.

### Other Failures

Transient send failures (5xx, network errors) are retried up to 3 times with exponential backoff. If all retries fail, the message is acknowledged and the error is logged — a single failed delivery does not block the outbound queue.

## Data Model

### ChatMapping

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique mapping identifier |
| `installation_id` | string (UUID) | Platform installation this mapping belongs to |
| `telegram_chat_id` | integer | Telegram chat identifier |
| `thread_id` | string (UUID) | Corresponding platform thread |
| `telegram_user_id` | integer | Telegram user ID of the chat initiator (DMs only) |
| `blocked_at` | timestamp | Set when the bot receives a "blocked by user" error; cleared on next inbound message |
| `created_at` | timestamp | When the mapping was created |

Index: `UNIQUE(installation_id, telegram_chat_id)` — used for all message routing lookups.

### InstallationState

Tracks per-installation polling state.

| Field | Type | Description |
|-------|------|-------------|
| `installation_id` | string (UUID) | Platform installation |
| `last_update_id` | integer | Last Telegram `update_id` successfully processed; used as `offset` on next `getUpdates` call |
| `updated_at` | timestamp | When the offset was last written |

## Dependencies

| Dependency | Usage |
|------------|-------|
| **Telegram Bot API** | Long polling (`getUpdates`), file download, message delivery (`sendMessage`, `sendPhoto`, etc.) |
| **Gateway → Threads** | `CreateThread`, `AddParticipant`, `SendMessage`, `GetUnackedMessages`, `AckMessages` |
| **Gateway → Files** | `UploadFile` (inbound media), `GetFileContent` (outbound media) |
| **Gateway → Notifications** | Subscribe to `thread_participant:{appId}` room for real-time message events |

## Architecture

```mermaid
graph TB
    subgraph "Telegram"
        TG[Telegram Bot API]
    end

    subgraph "Telegram Connector"
        Poller[Long Poll Loop<br/>per installation]
        Consumer[Message Consumer<br/>platform → telegram]
        DB[(PostgreSQL<br/>chat mappings)]
        Poller --> DB
        Consumer --> DB
    end

    subgraph Platform
        GW[Gateway]
        Threads
        Files
        Notifications
    end

    TG -->|getUpdates| Poller
    Poller -->|UploadFile, SendMessage| GW
    GW --> Threads
    GW --> Files

    Consumer -->|GetUnackedMessages, AckMessages| GW
    Consumer -->|GetFileContent| GW
    Consumer -->|sendMessage, sendPhoto, ...| TG
    Notifications -->|message.created| Consumer
```

## Flow

```mermaid
sequenceDiagram
    participant TG as Telegram
    participant TC as Telegram Connector
    participant GW as Gateway
    participant Th as Threads
    participant F as Files
    participant A as Agent

    Note over TG,TC: Inbound — user sends photo to bot
    TG-->>TC: getUpdates → photo message
    TC->>TG: getFile(file_id)
    TG-->>TC: file_path
    TC->>TG: Download file bytes
    TC->>GW: UploadFile(bytes, content_type)
    GW->>F: UploadFile
    F-->>GW: file UUID
    GW-->>TC: file UUID
    TC->>GW: SendMessage(thread_id, body: caption, files: [uuid])
    GW->>Th: SendMessage
    Th-->>TC: OK

    Note over A,Th: Agent processes message and responds
    A->>GW: SendMessage(thread_id, body: "Here is the result", files: [result_uuid])
    GW->>Th: SendMessage

    Note over TC,TG: Outbound — connector forwards response
    Th->>TC: message.created notification
    TC->>GW: GetUnackedMessages
    GW-->>TC: [message with body + files]
    TC->>TG: sendMessage("Here is the result")
    TC->>GW: GetFileContent(result_uuid)
    GW->>F: GetFileContent
    F-->>GW: file bytes
    GW-->>TC: file bytes
    TC->>TG: sendDocument(file bytes)
    TC->>GW: AckMessages
```

## Constraints

- The only body transformation applied to outbound messages is inline markdown image extraction (`![alt](url)` → `sendPhoto`). All other text formatting is sent as-is — agents should be instructed via their system prompt to produce plain text if Telegram markdown rendering is not desired.
- Group chats are not supported in the initial version — only direct messages to the bot.
- Recurrence and scheduling are not in scope — those are handled by the [Reminders](reminders.md) app.
- Telegram file size limits apply: 50 MB for most file types uploaded via bots.
- The connector does not support multiple agents per installation — one `agent_id` per installation.
