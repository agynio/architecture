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
| **SendMessage** | Send a message to a thread (text and/or file references) |
| **GetThreads** | List threads with pagination |
| **GetMessages** | List messages in a thread with pagination |
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

## Message Acknowledgment

`GetMessages` is a read-only operation — it does not change message state. `AckMessages` is a separate call where a participant explicitly marks messages as processed.

This separation handles crash recovery: a consumer can read messages, process them, and only acknowledge after successful processing. If the consumer crashes before acknowledging, the messages remain unacknowledged and can be re-read.

Unacknowledged messages per participant enable:
- Unread message counts in the UI (via [Chat](chat.md) service).
- The Agents orchestrator detecting threads with pending work.
- Channels detecting outbound messages to deliver to external apps.
