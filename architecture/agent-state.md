# Agent State (APSS)

## Overview

The Agent Persistent State Service (APSS) stores agent conversation context for long-term persistence. It uses `conversationId` as the primary routing key (threadId/nodeId mapping is handled outside the service).

## gRPC API

Defined in `agynio/api` at `proto/agynio/api/agent_state/v1/agent_state.proto`.

### Conversation Messages

| RPC | Description |
|-----|-------------|
| `AppendConversationMessages` | Append a batch of messages |
| `ListConversationMessages` | List with role/kind filtering and pagination (oldest → newest) |
| `ListConversationMessageIds` | List message IDs with pagination |
| `ReplaceConversationMessagesRange` | Replace a contiguous range with references |
| `DeleteConversationMessagesRange` | Delete a contiguous range |
| `GetConversation` | Get conversation metadata (count, timestamps) |

### Context Snapshots

Materialized copies of the exact context sent to an LLM call. Guarantees reproducibility after trims/edits.

| RPC | Description |
|-----|-------------|
| `CreateSnapshot` | Create from an ordered list of message IDs |
| `ListSnapshotMessages` | List materialized messages in a snapshot |
| `ListSnapshotMessageIds` | List message IDs in a snapshot |

## Data Model

### Message

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Server-assigned |
| `conversation_id` | string | Routing key |
| `role` | enum | `user`, `assistant`, `tool` |
| `kind` | string | Free-form tag (`system`, `summary`, `memory`, `tool_output`) |
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

### Context Snapshot

| Field | Type |
|-------|------|
| `snapshot_id` | string (UUID) |
| `conversation_id` | string |
| `created_at` | timestamp |

Each snapshot item includes a `body_sha256` integrity hash.
