# Threads

## Overview

Threads is the messaging service for conversations between participants. It stores messages, tracks participants, and provides message acknowledgment. Threads is participant-type-agnostic — it identifies participants by ID and applies the same behavior regardless of whether the participant is a user, an agent, or a channel.

Business logic (chat UX, agent processing, channel integration) is implemented by services built on top of Threads.

## Interface

| Method | Description |
|--------|-------------|
| **CreateThread** | Create a new thread with initial participants |
| **ArchiveThread** | Archive a thread (soft-delete) |
| **AddParticipant** | Add a participant to an existing thread |
| **SendMessage** | Send a message to a thread (text and/or file references). Creates a `MessageRecipient` row per recipient and publishes a `message.created` notification to each recipient's room |
| **GetThreads** | List threads with pagination |
| **GetMessages** | List messages in a thread with pagination. Read-only — does not change acknowledgment state |
| **GetUnackedMessages** | List unacknowledged messages for a participant across all threads |
| **AckMessages** | Acknowledge messages as processed by a participant |

## Data Model

### Thread

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique thread identifier |
| `participants` | list | Participants in the thread |
| `status` | enum | `active`, `archived` |
| `created_at` | timestamp | Creation time |
| `updated_at` | timestamp | Last modification time |

### Participant

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Participant identifier |
| `joined_at` | timestamp | When the participant joined |

### Message

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique message identifier |
| `thread_id` | string (UUID) | Parent thread |
| `sender_id` | string (UUID) | Participant who sent the message |
| `body` | string | Text content |
| `files` | list of string (UUID) | Referenced file IDs (may be empty). See [Media API](media.md#api) |
| `created_at` | timestamp | When the message was sent |

### MessageRecipient

Tracks acknowledgment state per participant per message. Created by `SendMessage` — one row per recipient (all thread participants except the sender).

| Field | Type | Description |
|-------|------|-------------|
| `message_id` | string (UUID) | FK to Message |
| `thread_id` | string (UUID) | Denormalized for query efficiency |
| `participant_id` | string (UUID) | Recipient |
| `acked_at` | timestamp (nullable) | NULL = unacknowledged |

Index: `(participant_id, acked_at)` — supports the cross-thread unacked query.

## Message Acknowledgment

`GetMessages` is a read-only operation — it does not change acknowledgment state. `AckMessages` is a separate call where a participant explicitly marks messages as processed.

This separation handles crash recovery: a consumer can read messages, process them, and only acknowledge after successful processing. If the consumer crashes before acknowledging, the messages remain unacknowledged and are returned by the next `GetUnackedMessages` call.

`GetUnackedMessages(participantId)` returns all unacknowledged messages for a participant across all threads. This enables consumers that participate in many threads (e.g., channels) to pull from a single endpoint.

## Notification Publishing

On `SendMessage`, Threads publishes a `message.created` event to the [Notifications](notifications.md) service for each recipient. The target room is `thread_participant:{participantId}`. Each consumer subscribes to its own room — one subscription regardless of how many threads it participates in.

Consumers combine notifications with pull to avoid duplicates — see [Consumer Sync Protocol](notifications.md#consumer-sync-protocol).

## Non-Participant Senders

Threads allows identities of type `app` to send messages without being thread participants. This supports [write-only apps](apps.md#write-only-apps) (e.g., Reminders) that post to threads but do not need to receive notifications or acknowledge messages.

When `SendMessage` is called with a `sender_id` whose `identity_type` is `app`:

1. The sender is **not** required to be a thread participant.
2. The message is created with the app's `sender_id`.
3. `MessageRecipient` rows are created for **all** thread participants (since the sender is not a participant, no participant is excluded from the recipient list).
4. Notifications are published to all participants' rooms.

Authorization is checked via [OpenFGA](authz.md) — the app must have `thread:write` permission. For cluster-scoped apps, this permission covers all threads in the platform. See [Apps — Permissions](apps.md#permissions).

