# Threads

## Overview

Threads is the messaging service for conversations between participants. A single thread can include multiple participants — both humans and agents.

## Interface

| Method | Description |
|--------|-------------|
| **CreateThread** | Create a new thread with initial participants |
| **ArchiveThread** | Archive a thread (soft-delete) |
| **AddParticipant** | Add a participant (human or agent) to an existing thread |
| **SendMessage** | Send a message to a thread (text and/or file references) |
| **GetThreads** | List threads with pagination |
| **GetMessages** | List messages in a thread with pagination |

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
| `type` | enum | `human`, `agent` |
| `joined_at` | timestamp | When the participant joined |

### Message

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique message identifier |
| `thread_id` | string (UUID) | Parent thread |
| `sender_id` | string (UUID) | Participant who sent the message |
| `body` | string | Text content |
| `files` | list of string (UUID) | Referenced file IDs (may be empty). See [Media](media.md) |
| `tokens` | integer | Token count for this message. Computed once at creation via [Token Counting](token-counting.md) |
| `read_status` | map | Per-participant read status |
| `created_at` | timestamp | When the message was sent |

## Read Status

Messages include read status tracked per participant. This enables:
- Unread message counts in the UI.
- The Agents orchestrator detecting pending (unread-by-agent) messages.
