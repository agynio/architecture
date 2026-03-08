# Agent State (APSS)

## Overview

The Agent Persistent State Service (APSS) stores agent conversation context for long-term persistence. It provides a conversation-oriented API where `conversationId` is the primary routing key (threadId/nodeId mapping is handled outside the service).

**Proto:** `agynio/api` at `proto/agynio/api/agent_state/v1/agent_state.proto`

## gRPC API

### Conversation Messages

| RPC | Description |
|-----|-------------|
| `AppendConversationMessages` | Append a batch of messages to a conversation |
| `ListConversationMessages` | List messages with role/kind filtering and pagination (oldest → newest) |
| `ListConversationMessageIds` | List message IDs with pagination |
| `ReplaceConversationMessagesRange` | Replace a contiguous range of messages with references to other messages |
| `DeleteConversationMessagesRange` | Delete a contiguous range of messages |
| `GetConversation` | Get conversation metadata (message count, timestamps) |

### Context Snapshots

Snapshots are materialized copies of the exact context sent to an LLM call. They guarantee reproducibility even after trims or edits to the conversation.

| RPC | Description |
|-----|-------------|
| `CreateSnapshot` | Create a snapshot from an ordered list of message IDs |
| `ListSnapshotMessages` | List materialized messages in a snapshot |
| `ListSnapshotMessageIds` | List message IDs in a snapshot |

## Data Model

### Message

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Server-assigned |
| `conversation_id` | string | Routing key |
| `role` | enum | `user`, `assistant`, `tool` |
| `kind` | string | Free-form tag (e.g., `system`, `summary`, `memory`, `tool_output`) |
| `body` | oneof | `TextBody` or `ToolOutputBody` |
| `created_at` | timestamp | |

### Message Body

**TextBody:** `{ text: string }`

**ToolOutputBody:**
| Field | Type |
|-------|------|
| `tool_name` | string |
| `tool_call_id` | string |
| `output` | oneof: `ToolTextOutput` or `ToolImageOutput` |
| `metadata` | map<string, string> |

### Conversation Metadata

| Field | Type |
|-------|------|
| `conversation_id` | string |
| `message_count` | int64 |
| `updated_at` | timestamp |

### Context Snapshot

| Field | Type |
|-------|------|
| `snapshot_id` | string (UUID) |
| `conversation_id` | string |
| `created_at` | timestamp |

Each snapshot item includes a `body_sha256` integrity hash for the materialized body copy.
