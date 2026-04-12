# Metering Service

## Overview

The Metering Service is the single store for all platform usage data. It ingests usage records from platform services, stores them in PostgreSQL, and exposes a query API for the console.

All observable and billable resource consumption flows through the Metering Service: LLM token usage, compute (CPU and RAM) consumed by workloads, and platform entity counts (threads, messages).

## Motivation

- **Observability**: org admins need realtime visibility into resource consumption, broken down by agent, model, and identity.
- **Billing**: usage data must be accurate across billing periods with no gaps due to service restarts or workload crashes.
- **Unified model**: discrete events (LLM calls, thread creation) and continuous resource consumption (compute) share the same storage and query path.

## Responsibilities

| Responsibility | Description |
|----------------|-------------|
| **Ingest** | Accept batches of usage records from platform services via gRPC |
| **Deduplicate** | Drop duplicate records by idempotency key |
| **Store** | Persist records in PostgreSQL, partitioned by timestamp |
| **Query** | Aggregate usage over time ranges and label dimensions |

## Ingestion Interface

Defined in `agynio/api` at `proto/agynio/api/metering/v1/metering.proto`.

### Record

Accepts a batch of usage records. Producers call this fire-and-forget — the operation that generated the usage (an LLM response, a workload sample tick) is not held waiting for the metering write to complete.

**Request:**

| Field | Type | Description |
|-------|------|-------------|
| `records` | repeated UsageRecord | Batch of usage records to ingest |

**UsageRecord:**

| Field | Type | Description |
|-------|------|-------------|
| `org_id` | string (UUID) | Organization that owns the usage |
| `idempotency_key` | string | Unique key for this record emission — see [Deduplication](#deduplication) |
| `producer` | string | Service that emitted the record (e.g., `"llm-proxy"`, `"orchestrator"`, `"threads"`) |
| `timestamp` | Timestamp | When the usage occurred |
| `labels` | map<string, string> | Dimensional metadata — see [Labels](#labels) |
| `unit` | Unit | Unit of measurement |
| `value` | int64 | Measured quantity in the given unit, scaled by 10^6 (micro-units). e.g., 1.5 GB-seconds = `1500000`, 1000 tokens = `1000000000` |

**Unit:**

| Value | Description |
|-------|-------------|
| `TOKENS` | Token count |
| `CORE_SECONDS` | CPU core × seconds |
| `GB_SECONDS` | Gigabyte × seconds |
| `COUNT` | Discrete event count |

### Labels

Labels carry dimensional metadata. The Metering Service stores and indexes them without interpreting their meaning. Producers include only the labels they have context for.

| Label | Description |
|-------|-------------|
| `resource_id` | UUID of the specific resource associated with the usage |
| `resource` | Resource type (e.g., `model`, `workload`, `thread`, `message`) |
| `identity_id` | UUID of the identity that caused the usage |
| `identity_type` | Type of identity: `agent`, `user`, `app` |
| `thread_id` | Thread associated with the usage (present when the producer has thread context) |
| `kind` | Subtype discriminator (e.g., `input`, `cached`, `output`, `ram`, `thread`, `message`) |
| `status` | Outcome of the operation (e.g., `success`, `failed`) |

## Deduplication

`usage_events` is partitioned by month on `timestamp`. The `UNIQUE(idempotency_key, month)` constraint is expressed within each partition. Since both `timestamp` and the idempotency key are derived from `interval_start`, duplicate submissions of the same key always land in the same monthly partition — the constraint provides global uniqueness.

On each ingest record:

```sql
INSERT INTO usage_events ...
ON CONFLICT (idempotency_key, month)
DO UPDATE SET value = EXCLUDED.value
```

**Crash scenario:** if a producer emits a record, the Metering Service stores it, but the producer crashes before updating its own `last_sampled_at` — on restart the producer recomputes the interval from the same `interval_start` but with a later `interval_end`, producing a larger value. The upsert updates the existing row in place. Once `last_sampled_at` is updated after a successful emit, subsequent cycles produce new keys and no conflict occurs.

## Query API

The Metering Service exposes a query API for aggregating usage over time ranges and label dimensions.

Queries always scope to `org_id`. Additional label filters narrow the result set. A `group_by` label key returns per-dimension breakdowns — for example, total input tokens grouped by `identity_id` for the per-agent usage view.

The [Gateway](gateway.md) exposes the query API to authenticated callers. Access is scoped to the caller's organization.

## Data Model

### UsageEvent

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Unique record identifier |
| `org_id` | UUID | Organization that owns the usage |
| `idempotency_key` | string | Deduplication key — see [Deduplication](#deduplication) |
| `producer` | string | Service that emitted the record |
| `timestamp` | timestamp | When the usage occurred. For sampled resources, this is the start of the sampled interval — aligned with the idempotency key and the partition key |
| `month` | timestamp (generated) | `date_trunc('month', timestamp)` — partition key and part of the dedup unique constraint |
| `unit` | Unit | Unit of measurement |
| `value` | NUMERIC(20, 6) | Measured quantity in the given unit. Converted from the wire int64 (÷ 10^6) on ingestion |
| `resource_id` | UUID (nullable) | Label: UUID of the specific resource |
| `resource` | string (nullable) | Label: resource type |
| `identity_id` | UUID (nullable) | Label: identity that caused the usage |
| `identity_type` | string (nullable) | Label: type of identity |
| `thread_id` | UUID (nullable) | Label: associated thread |
| `kind` | string (nullable) | Label: subtype discriminator |
| `status` | string (nullable) | Label: operation outcome |

Label columns are denormalized from the `labels` map for query performance. The table is partitioned by month on the generated `month` column, bounding time-range scans regardless of total history.

### Indexes

| Index | Purpose |
|-------|---------|
| `UNIQUE(idempotency_key, month)` | Deduplication — enforced per partition |
| `(org_id, timestamp)` | Base partition-local scan |
| `(org_id, unit, timestamp)` | Base query scan |
| `(org_id, identity_id, unit, timestamp)` | Per-identity queries |
| `(org_id, resource_id, unit, timestamp)` | Per-resource queries |

## Classification

| Aspect | Detail |
|--------|--------|
| **Plane** | Data |
| **API** | gRPC — `MeteringService.Record` (ingestion) + query methods |
| **State** | PostgreSQL, partitioned by month |
| **Scaling** | Scales with ingestion volume and active workload count |
| **Failure impact** | Temporary loss drops incoming usage records for the outage window; existing data remains queryable after recovery |
