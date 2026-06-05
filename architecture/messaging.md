# Messaging

## Overview

The platform's services communicate via two complementary mechanisms:

1. **Synchronous RPC** (gRPC over Istio) — for request/response, reads, and commands where the caller needs immediate confirmation. Existing pattern, no change.
2. **Asynchronous events** (NATS JetStream) — for state-change notifications between services, where the producer publishes a fact and any number of consumers react independently. This document defines the contract.

The event bus is the platform's source of truth for *what changed*. The producing service's database remains the source of truth for *current state*. Consumers receiving an event re-read the producer's state to act on it — they do not treat the event payload as the authoritative data.

The [Notifications](notifications.md) service is a separate primitive — fire-and-forget Redis pub/sub for *client-facing* real-time updates (Socket.IO to browsers). The two coexist:

| Concern | Mechanism |
|---|---|
| Service ↔ service durable state-change events | **NATS JetStream (this document)** |
| Service → client (browser) real-time UI updates | [Notifications](notifications.md) |
| Service ↔ service request/response | [gRPC](api-contracts.md) over Istio |

## Transport: NATS JetStream

The event bus is implemented as a single NATS server with JetStream enabled, deployed in-cluster via Helm.

| Aspect | Detail |
|---|---|
| **Software** | [NATS](https://nats.io) with JetStream persistence |
| **Deployment** | Helm chart (`nats-io/nats`) |
| **Persistence** | Local PVC per replica (file storage) |
| **Topology** | Single instance for dev/staging; 3-node cluster for production |
| **Clients** | Official [`nats.go`](https://github.com/nats-io/nats.go) SDK in producers and consumers |
| **Network** | Istio mTLS in-cluster; not exposed outside the cluster |
| **Footprint** | ~20MB RSS idle per instance; scales with stream retention |

NATS was chosen for:
- Lightweight operational footprint compared to Kafka.
- Native at-least-once delivery via durable JetStream streams and consumers.
- Subject hierarchy with wildcards — natural fit for the platform's domain decomposition.
- No external coordinator (no ZooKeeper, no JVM).
- Native Go client; matches the platform's primary language.

## Subject Naming Convention

Every event subject follows the same structure. Subjects are routing primitives; they are hard to change once published. Producers must follow this convention exactly.

```
agyn.<producer-service>.<entity>.<past-tense-action>
```

| Segment | Rules | Examples |
|---|---|---|
| `agyn.` | Fixed prefix. Reserves the namespace for platform events. Do not use in subject filters that aren't agyn-owned | — |
| `<producer-service>` | Lowercase, one word, matches the owning service's name. No version segment here | `groups`, `networks`, `users`, `agents`, `egress` |
| `<entity>` | Singular noun naming the resource type within the producer. Lowercase, snake_case if multi-word | `membership`, `resource`, `tunnel`, `group`, `egress_rule` |
| `<past-tense-action>` | Verb describing what happened. Past tense — events are facts about the past, not commands | `added`, `removed`, `created`, `updated`, `deleted`, `granted`, `revoked`, `online`, `offline` |

### Examples

```
agyn.groups.membership.added
agyn.groups.membership.removed
agyn.groups.group.deleted
agyn.networks.access.granted
agyn.networks.access.revoked
agyn.networks.tunnel.online
agyn.networks.tunnel.offline
agyn.networks.private_resource.deleted
```

### Wildcards

Consumers may subscribe to broader patterns using NATS wildcards:

- `*` matches a single segment
- `>` matches one or more segments

Common subscription patterns:

```
agyn.groups.membership.>            # all membership events from Groups
agyn.groups.>                       # everything Groups publishes
agyn.networks.tunnel.*              # tunnel online + offline only
agyn.*.access.>                     # access changes from any producer
```

### Anti-patterns

- **Imperative subject names.** `agyn.groups.member.add` is a command, not an event. Use `member.added`.
- **Mixed grammatical tense.** `created` and `creating` mean different things; stick to past tense uniformly.
- **Service version in subject by default.** `agyn.groups.v1.membership.added` is reserved for breaking schema changes (see [Versioning](#versioning)). New events under existing services do not carry a version.
- **Subjects encoding payload state.** `agyn.networks.tunnel.online.<network_id>` is wrong. Identifiers go in the payload, not the subject. The subject is the *type* of event, not its parameters.

## Event Envelope

The published message has two parts: **headers** (metadata) and **payload** (the domain event).

### Headers

Every published event carries the following NATS message headers. Producers set them at publish time; consumers may read them but must not modify them.

| Header | Value | Purpose |
|---|---|---|
| `Nats-Msg-Id` | UUID v4 | Deduplication key. NATS drops duplicates within its `duplicate_window` (default 2 min). Same value as `Agyn-Event-Id` |
| `Agyn-Event-Id` | UUID v4 | Application-level event identifier. Stable across redeliveries. Consumers may use this for explicit dedup if not relying on idempotent-by-construction handlers |
| `Agyn-Occurred-At` | RFC 3339 timestamp | When the underlying state change happened in the producer (e.g., DB commit time), not when the event was published. May be earlier than NATS arrival time |
| `Agyn-Producer` | string | Producing service identifier — `<service-name>` (e.g., `groups-service`, `networks-service`) |
| `Agyn-Schema` | string | Fully-qualified proto message name (e.g., `agyn.groups.v1.GroupMembershipAddedEvent`) — lets consumers route by schema independently of subject |
| `Agyn-Trace-Id` | string (optional) | [OpenTelemetry](tracing.md) trace ID of the operation that produced this event. Lets consumers continue the trace into their handler |
| `Agyn-Span-Id` | string (optional) | OpenTelemetry span ID of the publish operation |

Custom domain-specific headers may be added with the prefix `Agyn-X-`. Do not add custom headers without that prefix.

### Payload

The payload is the proto-serialized bytes of the domain event message. Nothing else. Routing/metadata lives in headers; the payload is pure domain.

Why this split:
- Routing/filtering (`Agyn-Schema`, subjects) does not require parsing payload bytes.
- Debugging tools (NATS CLI, `nats stream view`) can inspect headers without proto knowledge.
- Polyglot consumers (non-Go services, ad-hoc scripts) can route on subject and headers; payload decoding is per-consumer.
- The schema stays small and focused on the domain.

## Payload Schemas

### Where they live

Per-producer proto files in `agynio/api`, co-located with the producer's gRPC contracts:

```
agynio/api/proto/agynio/api/
├── groups/v1/
│   ├── groups.proto          # gRPC service definitions
│   └── events.proto          # event payload messages
├── networks/v1/
│   ├── networks.proto
│   └── events.proto
├── users/v1/
│   ├── users.proto
│   └── events.proto
```

The producing service owns both its API and its events. Consumers import the producer's `events.proto`. Schemas are versioned and reviewed alongside gRPC contracts.

### Schema design rules

**Minimal — just enough to identify the entity that changed.** Consumers re-read source-of-truth via gRPC when they need full state. The event is a trigger, not the data.

```proto
// proto/agynio/api/groups/v1/events.proto
syntax = "proto3";
package agyn.groups.v1;

message GroupMembershipAddedEvent {
  string group_id    = 1;
  string entity_type = 2;  // "user" | "agent" | "app"
  string entity_id   = 3;
}

message GroupMembershipRemovedEvent {
  string group_id    = 1;
  string entity_type = 2;
  string entity_id   = 3;
}

message GroupDeletedEvent {
  string group_id        = 1;
  string organization_id = 2;
}
```

Notice what is NOT in the payload:
- Timestamps (in `Agyn-Occurred-At` header)
- Trace IDs (in `Agyn-Trace-Id` header)
- Producer info (in `Agyn-Producer` header)
- Event ID (in `Agyn-Event-Id` header)
- Full entity state (consumers fetch via gRPC if needed)

This keeps payloads small and prevents the temptation to use the event payload as a cache of the producer's state.

### Versioning

- **Additive changes** (new optional fields, new oneof cases): no version bump. Consumers using proto3 ignore unknown fields by default.
- **Removed / renamed fields**: avoid. If unavoidable, publish a new schema under a versioned subject (`agyn.groups.v2.membership.added`). Old subject publishes continue during a deprecation window; consumers subscribe to both.
- **Field numbers**: never re-use a removed field's number for a different meaning. Use `reserved 5;` in the proto when a field is removed.
- **Semantic changes** (same field, different meaning): treat as a removed/renamed field — bump the subject version.

The subject's version segment is only present when a breaking change has happened. Default subjects do not include `v1` or any version.

## Stream Configuration

Streams are declared centrally as part of the messaging infrastructure deployment, not by individual application services. Each producing service domain gets one stream that captures all of its events.

```yaml
streams:
  - name: AGYN_GROUPS
    subjects: ["agyn.groups.>"]
    retention: limits
    storage: file
    max_age: 7d
    max_bytes: 1Gi
    replicas: 1            # 3 in production
    duplicates: 2m

  - name: AGYN_NETWORKS
    subjects: ["agyn.networks.>"]
    retention: limits
    storage: file
    max_age: 7d
    max_bytes: 1Gi
    replicas: 1
    duplicates: 2m
```

| Field | Default | Meaning |
|---|---|---|
| `retention: limits` | — | Messages stay until max_age / max_bytes hit. Consumers can replay |
| `storage: file` | — | Persisted to disk; survives NATS restart |
| `max_age: 7d` | — | Replay window. Enough to recover from multi-day consumer outage; not enough to be a long-term log |
| `max_bytes` | `1Gi` | Hard cap per stream. Tune per producer if needed |
| `replicas` | `1` dev / `3` prod | JetStream cluster replicas for HA |
| `duplicates` | `2m` | Dedup window — NATS drops messages with a `Nats-Msg-Id` seen in the last 2 minutes |

Adding new event types under an existing producer does not require stream changes — they fall under the existing `agyn.<producer>.>` subject filter.

Stream lifecycle (creation, retention updates, replica changes) is managed via the platform's infrastructure-as-code, not at application-deploy time.

## Consumer Patterns

### Durable consumers

Every subscribing service creates a **durable consumer** in JetStream. The durable name encodes the service and its purpose:

```
<service>-<purpose>
```

Examples: `users-group-sync`, `apps-group-sync`, `orchestrator-group-sync`, `networks-group-cleanup`.

Durable consumers persist their read position in the stream. A subscriber crash followed by restart resumes from the last acknowledged message — no events are lost or skipped.

### Multiple replicas of one service

When a service runs multiple replicas (e.g., three Users service pods), all replicas use the **same durable consumer name**. NATS distributes messages across the consumer's active subscribers — each message is delivered to exactly one replica. This gives load-balancing across replicas without duplicate processing within the service.

### Multiple consuming services

Each consuming service uses a **distinct durable consumer name**. Every service receives its own copy of every event. This is fan-out across services with load-balancing within a service.

```
                  ┌─────────────────────────────────┐
                  │  agyn.groups.membership.added   │
                  └─────────────────────────────────┘
                                  │
        ┌─────────────────┬───────┴───────┬─────────────────┐
        ▼                 ▼               ▼                 ▼
  users-group-sync   apps-group-sync   orchestrator-    networks-group-
   (3 replicas,        (1 replica,      group-sync       cleanup
    load-balanced)      gets all)       (2 replicas,     (1 replica)
                                         load-balanced)
```

### Acknowledgement policy

Consumers use **explicit ack** (`AckPolicy: AckExplicit`). The consumer calls `msg.Ack()` only after successful handling. On failure, the consumer calls `msg.Nak()` (or simply lets the ack-wait timeout expire) and NATS redelivers after backoff.

This guarantees at-least-once delivery. Combined with idempotent handlers (see below), this gives effectively-once processing semantics.

### Dead-letter handling

Consumers configure `MaxDeliver` (default 5) on their durable consumer. After 5 failed redeliveries, NATS stops redelivering and moves the message to a per-stream dead-letter subject. The platform operator monitors dead-letter counts and investigates.

```go
cons, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Durable:       "users-group-sync",
    FilterSubject: "agyn.groups.>",
    AckPolicy:     jetstream.AckExplicit,
    AckWait:       30 * time.Second,
    MaxDeliver:    5,
})
```

## Idempotency

NATS JetStream provides **at-least-once** delivery. Consumers must tolerate seeing the same event more than once — through redelivery on consumer failure, network blips, or rebalancing across replicas.

### Preferred: idempotent by construction

The strongly preferred pattern. The handler re-reads source-of-truth and reconciles to the desired state. Running the handler N times produces the same end state as running it once.

Example — Groups membership sync in the Users service:

```go
func (s *UsersService) onGroupMembershipChanged(ctx context.Context, msg jetstream.Msg) error {
    event := &groupsv1.GroupMembershipAddedEvent{}
    if err := proto.Unmarshal(msg.Data(), event); err != nil {
        return msg.Ack()  // unparseable — drop; investigation needed
    }
    if event.EntityType != "user" {
        return msg.Ack()  // not our concern
    }

    // Re-read source of truth — do not trust the event payload as state
    groups, err := s.groupsClient.ListMemberGroups(ctx, event.EntityId)
    if err != nil {
        return msg.Nak()  // retry
    }

    devices, err := s.repo.ListUserDevices(ctx, event.EntityId)
    if err != nil {
        return msg.Nak()
    }

    desired := computeRoleAttributes(groups)
    for _, d := range devices {
        if err := s.zitiMgmt.PatchIdentityRoleAttributes(ctx, d.OpenZitiIdentityId, desired); err != nil {
            return msg.Nak()
        }
    }
    return msg.Ack()
}
```

The handler does not care whether this is the first delivery or the third. It computes desired state from source-of-truth and applies it. Duplicates are a no-op.

### Fallback: explicit dedup

Use only when reconciling from source-of-truth is not possible (e.g., side-effect-only handlers — sending an email, charging a credit card, writing an audit record).

Track `Agyn-Event-Id` in a `processed_events` table:

```sql
CREATE TABLE processed_events (
  event_id  uuid PRIMARY KEY,
  processed_at timestamptz NOT NULL DEFAULT now()
);
```

Handler skips events whose IDs are already in the table.

## When to Use Events vs RPC

Both event-bus messaging and synchronous RPC are first-class platform primitives. Choose based on the semantics of the interaction, not by reflex.

### Use events when

- The producer publishes a fact and does not care who reacts.
- Multiple consumers may want the same notification (fan-out).
- The reaction is not on the producer's request path (latency cost shifted to consumers).
- A failure in one consumer must not affect others.
- The producer should not know about the consumers (loose coupling).
- Eventual consistency is acceptable for the action being taken.

Examples: group membership changed, tunnel went offline, EgressRule attached to agent, workload status transitioned.

### Use RPC when

- The caller needs the response to complete its own operation (reads, lookups).
- Strong consistency is required at the moment of the call.
- Only one specific service can fulfill the request.
- Failure modes need to surface directly to the caller for retry / fallback decisions.

Examples: `Groups.ListMemberGroups`, `Apps.GetAppIdentity`, `ZitiManagement.CreateService`, `Authorization.Check`.

### Anti-pattern: events as command channel

Publishing an event with the intent that exactly one specific consumer act on it is not events — that's a command. Use RPC. Events are for state-change facts that any number of interested parties may react to.

## Documentation Pattern

### Central registry

[`architecture/messaging.md`](messaging.md) (this document) maintains the central catalog of all event subjects in the platform — see [Event Registry](#event-registry) below. Adding a new event subject requires updating this catalog.

### Producer service docs

Each producing service's architecture doc has an **Events Published** section enumerating its events:

```markdown
## Events Published

| Subject | Schema | Published when |
|---|---|---|
| `agyn.groups.membership.added` | `agyn.groups.v1.GroupMembershipAddedEvent` | `AddMember` RPC completes — DB row and OpenFGA tuple committed |
| `agyn.groups.membership.removed` | `agyn.groups.v1.GroupMembershipRemovedEvent` | `RemoveMember` RPC completes |
| `agyn.groups.group.deleted` | `agyn.groups.v1.GroupDeletedEvent` | `DeleteGroup` RPC completes — DB row removed, OpenFGA tuples deleted |
```

### Consumer service docs

Each consuming service's architecture doc has an **Events Consumed** section enumerating its subscriptions:

```markdown
## Events Consumed

| Subject filter | Durable consumer | Purpose |
|---|---|---|
| `agyn.groups.membership.>` | `users-group-sync` | PATCH user device identities to reflect group membership changes |
| `agyn.groups.group.deleted` | `users-group-cleanup` | Remove `group-<id>` attribute from user device identities for deleted groups |
```

## Operational Concerns

### Observability

NATS JetStream exposes Prometheus metrics for stream depth, consumer lag, redelivery counts, and per-consumer ack rates. Consumer-side, services emit OpenTelemetry spans for each handled message (correlated to the producer's span via `Agyn-Trace-Id`).

Key dashboards:
- Stream depth per stream (alerting when growing unboundedly)
- Consumer lag per durable consumer (alerting when growing unboundedly)
- Dead-letter count per stream (alerting on any non-zero count)
- Handler latency per consumer

### Replay

JetStream's `max_age` retention enables replay. A new consumer can start from the beginning of the stream (`DeliverPolicy: DeliverAll`) and process all events still in retention. Used for:

- Onboarding a new consuming service into an existing event stream.
- Recovering from a multi-day consumer outage by replaying missed events.
- Backfilling data when a new feature needs to react to historical events.

After a consumer catches up, future events are delivered live.

### Schema discovery

The proto files in `agynio/api` are the source of truth for event schemas. Consumers import the producer's proto package. CI validates that producer proto changes are backward-compatible via `buf breaking` checks.

### Local development

Developers run NATS locally via `docker run -p 4222:4222 nats:latest -js` or via the platform's `docker-compose.dev.yml`. Services connect to `nats://localhost:4222` in dev. JetStream streams are auto-created by the messaging infrastructure helm chart in real clusters; for local dev a small bootstrap script creates them in the local NATS instance.

## Event Registry

The authoritative catalog of all event subjects published in the platform. Adding or removing a subject requires updating this table. Schemas are linked to the proto file in `agynio/api`.

| Subject | Producer | Schema | Description |
|---|---|---|---|
| `agyn.groups.membership.added` | [Groups](groups-service.md) | `agyn.groups.v1.GroupMembershipAddedEvent` | A member was added to a group |
| `agyn.groups.membership.removed` | [Groups](groups-service.md) | `agyn.groups.v1.GroupMembershipRemovedEvent` | A member was removed from a group |
| `agyn.groups.group.deleted` | [Groups](groups-service.md) | `agyn.groups.v1.GroupDeletedEvent` | A group was deleted; consuming services should clean up references |
| `agyn.networks.tunnel.online` | [Networks](networks-service.md) | `agyn.networks.v1.TunnelOnlineEvent` | A tunnel credential's identity gained an active OpenZiti session |
| `agyn.networks.tunnel.offline` | [Networks](networks-service.md) | `agyn.networks.v1.TunnelOfflineEvent` | A tunnel credential's identity lost its active OpenZiti session |
| `agyn.networks.access.granted` | [Networks](networks-service.md) | `agyn.networks.v1.PrivateResourceAccessGrantedEvent` | A private-resource access grant was created |
| `agyn.networks.access.revoked` | [Networks](networks-service.md) | `agyn.networks.v1.PrivateResourceAccessRevokedEvent` | A private-resource access grant was deleted |

## Related Architecture

- [Notifications](notifications.md) — client-facing real-time updates (Socket.IO), distinct from this event bus
- [API Contracts](api-contracts.md) — synchronous gRPC contracts
- [Tracing](tracing.md) — distributed tracing across event boundaries via `Agyn-Trace-Id`
- [Groups Service](groups-service.md) — first producer of group lifecycle events
- [Networks Service](networks-service.md) — producer of tunnel and private-resource access events
