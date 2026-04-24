# Threads

## Overview

Threads is the messaging service for conversations between participants. It stores messages, tracks participants, and provides message acknowledgment. Threads is participant-type-agnostic — it identifies participants by ID and applies the same behavior regardless of whether the participant is a user, an agent, or an app.

Business logic (chat UX, agent processing, app integration) is implemented by services built on top of Threads.

## Interface

| Method | Description |
|--------|-------------|
| **CreateThread** | Create a new thread with initial participants. Requires `organization_id` |
| **ArchiveThread** | Archive a thread (soft-delete) |
| **DegradeThread** | Mark a thread as degraded. Internal only — called by the [Agents Orchestrator](agents-orchestrator.md) when a thread cannot be recovered (persistent volume lost, hosting runner deprovisioned, or agent start failures exhausted — see [Start Decision](agents-orchestrator.md#start-decision)). Accepts a `reason` string. Idempotent — repeated calls on an already-degraded thread are a no-op |
| **AddParticipant** | Add a participant to an existing thread. Accepts an `identity_id` or a `@nickname` (resolved to `identity_id` internally). Accepts a `passive` flag — passive participants receive messages but do not trigger workload starts in the [Agents Orchestrator](agents-orchestrator.md) |
| **SendMessage** | Send a message to a thread (text and/or file references). Creates a `MessageRecipient` row per recipient and publishes a `message.created` notification to each recipient's room |
| **GetThreads** | List threads the caller participates in, with pagination |
| **ListOrganizationThreads** | List all threads in an organization with server-side sort, filter, and pagination. Requires `can_view_threads` on the organization. See [ListOrganizationThreads request shape](#listorganizationthreads-request-shape) |
| **GetMessages** | List messages in a thread with pagination. Read-only — does not change acknowledgment state. Accessible to thread participants and identities with `can_view_threads` on the thread's organization |
| **GetUnackedMessages** | List unacknowledged messages for a participant. Supports optional `thread_id` filter to scope results to a single thread |
| **AckMessages** | Acknowledge messages as processed by a participant |

## Data Model

### Thread

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique thread identifier |
| `organization_id` | string (UUID) | Organization that owns the thread |
| `participants` | list | Participants in the thread |
| `status` | enum | `active`, `archived`, `degraded` |
| `created_at` | timestamp | Creation time |
| `updated_at` | timestamp | Last modification time |

### Participant

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Participant identifier |
| `passive` | boolean | When `true`, the [Agents Orchestrator](agents-orchestrator.md) does not start a workload for this participant when there are unread messages. The participant is expected to consume messages via the API directly |
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

## Thread Status

| Status | Description |
|--------|-------------|
| `active` | Normal operating state. All operations permitted |
| `archived` | Soft-deleted by a user or application. No new messages accepted |
| `degraded` | Degraded and unrecoverable. No new messages accepted. Set by the [Agents Orchestrator](agents-orchestrator.md) via `DegradeThread` with a machine-readable reason such as `volume_lost`, `runner_deprovisioned`, or `agent_start_failures_exhausted` |

`SendMessage` returns an error for `archived` and `degraded` threads. All read operations (`GetMessages`, `GetUnackedMessages`) remain available on both statuses.

## ListOrganizationThreads request shape

The Console's Activity → Threads view backs onto this method. Thread lists can be large, so sort, filter, and pagination are server-side. Callers must not filter or sort across pages on the client. See [Console — Resource Lists](../product/console/console.md#resource-lists).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `organization_id` | string (UUID) | Yes | Authorization scope. Caller must hold `can_view_threads` on this organization |
| `filter.status_in` | list<Thread.Status> | No | Return only threads in these statuses (`active`, `archived`, `degraded`) |
| `filter.participant_id_in` | list<string (UUID)> | No | Return only threads where any of these identities is a participant |
| `filter.created_after` | timestamp | No | Return only threads with `created_at >= created_after` |
| `filter.created_before` | timestamp | No | Return only threads with `created_at < created_before` |
| `sort.field` | enum | No | One of `created`, `updated`, `message_count`, `status`. Default: `created` |
| `sort.direction` | enum | No | `asc` or `desc`. Default: `desc` |
| `page_token` | string | No | Opaque cursor returned by the previous response. Empty on the first page |
| `page_size` | int32 | No | Maximum items to return. Server enforces an upper bound |

Filters combine with AND; within a list field (`*_in`), values combine with OR. Changing `sort` or `filter` resets pagination — callers must discard any previous `page_token`.

The server applies a stable secondary sort by `id` (ascending) on every response, so ties on the primary sort field produce a deterministic order and pagination does not skip or duplicate rows.

Response items include every [Thread](#thread) field plus `message_count` (number of messages in the thread) and, for each participant, the resolved `@nickname` so the UI renders names, not IDs.

## Message Acknowledgment

`GetMessages` is a read-only operation — it does not change acknowledgment state. `AckMessages` is a separate call where a participant explicitly marks messages as processed.

This separation handles crash recovery: a consumer can read messages, process them, and only acknowledge after successful processing. If the consumer crashes before acknowledging, the messages remain unacknowledged and are returned by the next `GetUnackedMessages` call.

`GetUnackedMessages(participantId)` returns all unacknowledged messages for a participant across all threads. This enables consumers that participate in many threads (e.g., apps) to pull from a single endpoint.

## Notification Publishing

On `SendMessage`, Threads publishes a `message.created` event to the [Notifications](notifications.md) service for each recipient. The target room is `thread_participant:{participantId}`. Each consumer subscribes to its own room — one subscription regardless of how many threads it participates in.

Consumers combine notifications with pull to avoid duplicates — see [Consumer Sync Protocol](notifications.md#consumer-sync-protocol).

## Metering

The Threads service emits usage records to the [Metering Service](metering.md) on each thread or message creation.

| unit | value | labels | idempotency_key |
|------|-------|--------|-----------------|
| `COUNT` | 1 | resource_id=thread_id, resource=thread, kind=thread | thread_id |
| `COUNT` | 1 | resource_id=message_id, resource=message, kind=message, thread_id | message_id |

## Authorization

Thread access is enforced via [OpenFGA](authz.md). Each thread is an OpenFGA object (`thread:<id>`) with relations to its organization and its participant set.

| Relation | Who holds it |
|----------|-------------|
| `participant` | Identities explicitly added via `CreateThread` or `AddParticipant` |
| `can_read` | Participants; identities with `can_view_threads` on the thread's org (i.e., org owners) |
| `can_write` | Participants; app identities with `thread_write` on the org |
| `can_add_participant` | Participants; app identities with `participant_add` on the org |

**Tuple writes:**
- `CreateThread`: writes `organization:<org_id>, org, thread:<id>` and `identity:<id>, participant, thread:<id>` for each initial participant.
- `AddParticipant`: writes `identity:<id>, participant, thread:<id>`.

There are no tuple deletes when participants leave or when threads are archived (threads are soft-deleted; tuples become orphaned but harmless).

## Non-Participant Senders

Threads allows identities of type `app` to send messages without being thread participants. This supports [write-only apps](apps.md#write-only-apps) (e.g., Reminders) that post to threads but do not need to receive notifications or acknowledge messages.

When `SendMessage` is called with a `sender_id` whose `identity_type` is `app`:

1. The sender is **not** required to be a thread participant.
2. The message is created with the app's `sender_id`.
3. `MessageRecipient` rows are created for **all** thread participants (since the sender is not a participant, no participant is excluded from the recipient list).
4. Notifications are published to all participants' rooms.

Authorization is checked via [OpenFGA](authz.md) — the app must have `thread:write` permission. For cluster-scoped apps, this permission covers all threads in the platform. See [Apps — Permissions](apps.md#permissions).

