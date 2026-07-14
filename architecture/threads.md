# Threads

## Overview

Threads is the messaging service for conversations between participants. It stores messages, tracks participants, and provides message acknowledgment. Threads is participant-type-agnostic — it identifies participants by ID and applies the same behavior regardless of whether the participant is a user, an agent, or an app.

Business logic (chat UX, agent processing, app integration) is implemented by services built on top of Threads.

## Runtime Associations

Threads owns messages and participants. It does not own workload or storage
runtime state; those records are owned by [Runners](runners.md).

Associated workloads are queried from Runners with
`ListWorkloadsByThread(thread_id)`. Thread storage is queried from Runners with
`ListVolumesByThread(thread_id)`. When storage is shown from a workload context,
the Console passes the workload's `thread_id` to Runners and shows storage
associated with that thread. That storage is not an exact per-workload mounted
storage list unless a workload-scoped mount API provides that evidence.

## Interface

| Method | Description |
|--------|-------------|
| **CreateThread** | Create a new thread with initial participants. Requires `organization_id`. For each agent-class participant, additionally enforces the [Agent Availability Check](#agent-availability-check) and applies the [class → instance rewrite](#class-on-add-rewrite) |
| **ArchiveThread** | Archive a thread (soft-delete) |
| **DegradeThread** | Mark a thread as degraded. Internal only — called by the [Agents Orchestrator](agents-orchestrator.md) when a thread cannot be recovered (persistent volume lost, hosting runner deprovisioned, or agent start failures exhausted — see [Start Decision](agents-orchestrator.md#start-decision)). Accepts a `reason` string. Idempotent — repeated calls on an already-degraded thread are a no-op |
| **AddParticipant** | Add a participant to an existing thread. Accepts an `identity_id` or a `@nickname` (resolved to `identity_id` internally). If the resolved identity is an agent class, applies the [class → instance rewrite](#class-on-add-rewrite) and enforces the [Agent Availability Check](#agent-availability-check) |
| **SendMessage** | Send a message to a thread (text and/or file references). For each non-sender participant, delivers the message to that participant's consumer channel — `MessageRecipient` for users and apps, [inbox item](agent-instances.md#inbox) for agent instances — and publishes a `message.created` notification to the corresponding room. See [Message Delivery](#message-delivery) |
| **GetThreads** | List threads the caller participates in, with pagination |
| **ListOrganizationThreads** | List all threads in an organization with server-side sort, filter, and pagination. Requires `can_view_threads` on the organization. See [ListOrganizationThreads request shape](#listorganizationthreads-request-shape) |
| **GetMessages** | List messages in a thread with pagination. Read-only — does not change acknowledgment state. Accessible to thread participants and identities with `can_view_threads` on the thread's organization |
| **GetUnackedMessages** | List unacknowledged messages for a participant. Supports optional `thread_id` filter to scope results to a single thread |
| **GetUnackedMessageCounts** | Return a `map<thread_id, count>` of unacknowledged messages per thread for a participant, across every thread the participant is on. Self-only — `participant_id` must equal `caller.identity_id` |
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
| `id` | string (UUID) | Participant identity id. For agents this is always an [agent instance](agent-instances.md) id — classes are rewritten to instances on add |
| `joined_at` | timestamp | When the participant joined |

Threads stores identities, not classes. When a caller adds an agent class as a participant, Threads rewrites the stored id to a fresh instance's id (see [Class-on-Add Rewrite](#class-on-add-rewrite)). From the thread's perspective, an agent participant is always an instance.

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

Tracks acknowledgment state for **user and app** participants. Created by `SendMessage` — one row per non-sender participant of type `user` or `app`. Agent-instance participants receive [inbox items](agent-instances.md#inbox) instead; see [Message Delivery](#message-delivery).

| Field | Type | Description |
|-------|------|-------------|
| `message_id` | string (UUID) | FK to Message |
| `thread_id` | string (UUID) | Denormalized for query efficiency |
| `participant_id` | string (UUID) | Recipient identity id (`user` or `app`) |
| `acked_at` | timestamp (nullable) | NULL = unacknowledged |

Index: `(participant_id, acked_at)` — supports the cross-thread unacked query.

## Message Delivery

`SendMessage` delivers to each non-sender participant according to the participant's identity type:

| Participant type | Delivery |
|------------------|----------|
| `user`, `app` | Insert `MessageRecipient` row; publish `message.created` to `thread_participant:{participant_id}` |
| `agent_instance` | Insert an [inbox item](agent-instances.md#inbox) on the instance's inbox with `source_kind=thread`; publish `message.created` to `instance_inbox:{instance_id}` |

The two paths are mutually exclusive per participant. `GetUnackedMessages` / `AckMessages` remain the surface for `user` and `app` consumers; agent-instance consumers use `GetUnackedInboxItems` / `AckInboxItems` on the [Agents Service](agents-service.md) inbox API. Threads does not expose the inbox surface itself — it only writes into it during fan-out.

## Class-on-Add Rewrite

`CreateThread` and `AddParticipant` accept either an instance id, a user/app id, or an agent class id (directly or resolved from a class `@nickname`). When the resolved identity is an agent class:

1. Threads calls the [Agents Service](agents-service.md) to create a fresh instance of the class in the thread's organization.
2. The [Agent Availability Check](#agent-availability-check) is performed against the class (not the instance) — the caller must satisfy `can_initiate` on the class.
3. The participant is stored with the newly-created instance's id, not the class's id.
4. The response returns the resolved participant id so the caller can address the specific instance for follow-up (e.g., `@bob#7a2f`).

Adding an **existing instance** id skips instance creation but still enforces the same availability check against the instance's class. Adding the caller's own running instance (agent-to-agent delegation) reuses that instance — no new one is spawned.

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

`GetUnackedMessageCounts(participantId)` returns a `map<thread_id, count>` of unacknowledged messages per thread, across every thread the participant is on. It is the count-only complement of `GetUnackedMessages` for callers that need per-thread totals (e.g., the [Chat](chat.md) service rendering unread badges) and would otherwise have to paginate full message bodies just to count them. The same `(participant_id, acked_at)` index that backs `GetUnackedMessages` serves this query.

## Notification Publishing

On `SendMessage`, Threads publishes a `message.created` event to the [Notifications](notifications.md) service for each recipient. The target room depends on the recipient's identity type:

| Recipient type | Room |
|----------------|------|
| `user`, `app` | `thread_participant:{participant_id}` |
| `agent_instance` | `instance_inbox:{instance_id}` |

Each consumer subscribes to its own room. Users and apps subscribe to their `thread_participant` room across all their threads; agent-instance consumers subscribe to their inbox room.

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

**Per-participant agent check.** `CreateThread` and `AddParticipant` additionally check `Check(caller, can_initiate, agent:<class_id>)` for every participant being added whose `identity_type` is `agent` (a class) or `agent_instance` (resolved through the instance's class). See [Agent Availability Check](#agent-availability-check). The thread-level `can_add_participant` check still applies; both must pass.

**Tuple writes:**
- `CreateThread`: writes `organization:<org_id>, org, thread:<id>` and `identity:<id>, participant, thread:<id>` for each initial participant.
- `AddParticipant`: writes `identity:<id>, participant, thread:<id>`.

There are no tuple deletes when participants leave or when threads are archived (threads are soft-deleted; tuples become orphaned but harmless).

## Agent Availability Check

When `CreateThread` or `AddParticipant` adds an identity whose `identity_type` is `agent` (a class) or `agent_instance`, Threads performs a second OpenFGA check in addition to the thread-level participant-add check. The check is always against the **class**:

```
Check(caller, can_initiate, agent:<class_id>) → allowed
```

For `agent_instance` inputs, Threads resolves the class id from the instance record via the [Agents Service](agents-service.md).

If the check fails, the operation is rejected with a permission error. The check semantics derive from the agent's [availability](agents-service.md#availability) value and its [role assignments](agents-service.md#roles):

- `internal` — any org member satisfies `can_initiate`.
- `private` — only identities holding `owner`, `maintainer`, or `participant` on the agent satisfy `can_initiate`.

The check is always against the **caller**, never against existing thread participants. Sharing a thread with an agent does not transitively grant the right to add the agent (or any other private agent) elsewhere. App identities holding `participant:add` or `thread:create` on the organization are subject to the same check — those org-level permissions are not enough to bypass per-agent gating. To make a private agent addable by an app (for example, a Telegram connector adding the agent to an inbound chat), an agent `owner` must grant the app a role on the agent via `SetAgentRole`.

Identity-type resolution reuses the same [Identity](identity.md) lookup that resolves `@nickname` to `identity_id`. Threads issues `BatchGetIdentityTypes` for all participants being added, then issues `Check` calls for participants of type `agent` or `agent_instance` (using the class id in the latter case).

## Non-Participant Senders

Threads allows identities of type `app` to send messages without being thread participants. This supports [write-only apps](apps.md#write-only-apps) (e.g., Reminders) that post to threads but do not need to receive notifications or acknowledge messages.

When `SendMessage` is called with a `sender_id` whose `identity_type` is `app`:

1. The sender is **not** required to be a thread participant.
2. The message is created with the app's `sender_id`.
3. Delivery rows are created for **all** thread participants (since the sender is not a participant, no participant is excluded) per the [Message Delivery](#message-delivery) rules — `MessageRecipient` for users/apps, inbox items for agent instances.
4. Notifications are published to all participants' rooms.

Authorization is checked via [OpenFGA](authz.md) — the app must have `thread:write` permission. For cluster-scoped apps, this permission covers all threads in the platform. See [Apps — Permissions](apps.md#permissions).
