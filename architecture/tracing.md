# Tracing

## Overview

The Tracing service ingests, stores, and queries span data. It implements the standard [OpenTelemetry](https://opentelemetry.io/) Collector `TraceService/Export` gRPC interface with one behavioral extension: **upsert semantics for in-progress spans**. Standard OpenTelemetry assumes spans are exported once after completion. Agyn needs visibility into ongoing agent operations, so producers export the same span multiple times — first while in progress, then again when completed. The Tracing service detects duplicates by `(trace_id, span_id)` and patches the existing record instead of inserting a new one.

Tracing captures the **full LLM call context** — the complete request body (all messages sent to the model) for each LLM call. This makes tracing the primary source for debugging and inspecting what the model saw at each step. Due to the volume of data, tracing has shorter retention than conversation records in [Threads](threads.md). See [Agent State — Isolation](agent/state.md#isolation) for how this fits into the platform's data separation model.

## Responsibilities

- **Ingest** spans via the standard OTLP gRPC interface with upsert semantics.
- **Store** spans in PostgreSQL.
- **Query** traces and spans for the observability UI (list, filter, detail).
- **Push notifications** on span creation and update so the UI receives real-time changes without polling.

## Integration

Tracing is an **optional** dependency for agents. Agents can run without a tracing endpoint configured.

## Ingestion

### Protocol

The Tracing service implements the standard OTLP Collector gRPC interface:

```
opentelemetry.proto.collector.trace.v1.TraceService/Export
```

| Field | Type | Description |
|-------|------|-------------|
| `resource_spans` | repeated `ResourceSpans` | Standard OTLP envelope — resources → scopes → spans |

Standard OTel SDK exporters (Go, Python, JS, etc.) can point their OTLP gRPC exporter at the Tracing service endpoint without modification.

### Upsert Semantics

For each span in the request, the service performs an upsert keyed on `(trace_id, span_id)`:

- **No existing record** → insert.
- **Existing record** → patch all fields with the incoming values.

This allows producers to export a span multiple times during its lifecycle. The first export creates the record; subsequent exports update it as the operation progresses (new attributes, events, status changes, end time).

### In-Progress Detection

A span is considered **in-progress** when `end_time_unix_nano = 0`. A non-zero `end_time_unix_nano` indicates the span is **completed**.

Producers set `end_time_unix_nano = 0` on intermediate exports and set the real end time on the final export.

### Response

The service returns the standard `ExportTraceServiceResponse`:

| Field | Type | Description |
|-------|------|-------------|
| `partial_success.rejected_spans` | int64 | Number of spans rejected (0 = fully accepted) |
| `partial_success.error_message` | string | Human-readable explanation if spans were rejected |

## Data Model

### Span (stored)

Each span is stored as a row in PostgreSQL. The schema follows the [OTel Span](https://buf.build/opentelemetry/opentelemetry/docs/main:opentelemetry.proto.trace.v1#opentelemetry.proto.trace.v1.Span) data model.

| Field | Type | Description |
|-------|------|-------------|
| `trace_id` | bytes (16) | Trace identifier |
| `span_id` | bytes (8) | Span identifier |
| `trace_state` | string | W3C trace-context trace state |
| `parent_span_id` | bytes (8) | Parent span identifier (empty for root spans) |
| `flags` | uint32 | W3C trace flags + OTel span flags |
| `name` | string | Operation name |
| `kind` | enum | `UNSPECIFIED`, `INTERNAL`, `SERVER`, `CLIENT`, `PRODUCER`, `CONSUMER` |
| `start_time_unix_nano` | fixed64 | Span start time (nanoseconds since epoch) |
| `end_time_unix_nano` | fixed64 | Span end time (0 = in-progress) |
| `attributes` | repeated KeyValue | Span attributes |
| `events` | repeated Event | Timestamped span events |
| `links` | repeated Link | Links to other spans |
| `status` | Status | Status code (`UNSET`, `OK`, `ERROR`) and message |
| `dropped_attributes_count` | uint32 | Number of dropped attributes |
| `dropped_events_count` | uint32 | Number of dropped events |
| `dropped_links_count` | uint32 | Number of dropped links |

**Resource** and **InstrumentationScope** metadata from the OTLP envelope are stored alongside the span (flattened or as JSONB columns — implementation detail).

Query API responses use the standard OTel envelope hierarchy (`ResourceSpans` -> `ScopeSpans` -> `Span`) to return Resource and InstrumentationScope context alongside each span. This avoids custom wrapper types and follows the same pattern as Jaeger v3 QueryService.

Primary key: `(trace_id, span_id)`.

### Indexes

| Index | Purpose |
|-------|---------|
| `(trace_id)` | Fetch all spans for a trace |
| `(start_time_unix_nano)` | Time-range queries and ordering |
| `(parent_span_id)` | Tree reconstruction |

## Query API

Defined in `agynio/api` at `proto/agynio/api/tracing/v1/tracing.proto`.

| RPC | Description |
|-----|-------------|
| `ListSpans` | Paginated span listing with filters |
| `GetSpan` | Single span by `(trace_id, span_id)` |
| `GetTrace` | All spans for a `trace_id` |

### ListSpans

Returns a paginated list of spans matching the provided filters.

**Request:**

| Field | Type | Description |
|-------|------|-------------|
| `filter` | `SpanFilter` | Filter criteria (all optional, AND-combined) |
| `page_size` | int32 | Maximum number of spans to return |
| `page_token` | string | Pagination cursor from previous response |
| `order_by` | enum | `START_TIME_DESC` (default), `START_TIME_ASC` |

**SpanFilter:**

| Field | Type | Description |
|-------|------|-------------|
| `trace_id` | bytes | Filter by trace |
| `parent_span_id` | bytes | Filter by parent span |
| `name` | string | Filter by operation name (exact match) |
| `kind` | SpanKind | Filter by span kind |
| `start_time_min` | fixed64 | Start time lower bound (inclusive) |
| `start_time_max` | fixed64 | Start time upper bound (inclusive) |
| `in_progress` | optional bool | `true` = only in-progress, `false` = only completed, unset = both |

**Response:**

| Field | Type | Description |
|-------|------|-------------|
| `resource_spans` | repeated `ResourceSpans` | Matching spans in the standard OTel envelope hierarchy (ResourceSpans -> ScopeSpans -> Span). Spans sharing a Resource and InstrumentationScope are grouped. Page size applies to total Span count. |
| `next_page_token` | string | Cursor for next page (empty = no more results) |

### GetSpan

Returns a single span.

**Request:**

| Field | Type | Description |
|-------|------|-------------|
| `trace_id` | bytes (16) | Trace identifier |
| `span_id` | bytes (8) | Span identifier |

**Response:**

| Field | Type | Description |
|-------|------|-------------|
| `resource_spans` | repeated `ResourceSpans` | The span in its OTel envelope (one ResourceSpans -> one ScopeSpans -> one Span) |

### GetTrace

Returns all spans belonging to a trace.

**Request:**

| Field | Type | Description |
|-------|------|-------------|
| `trace_id` | bytes (16) | Trace identifier |

**Response:**

| Field | Type | Description |
|-------|------|-------------|
| `resource_spans` | repeated `ResourceSpans` | All spans in the trace, grouped by Resource and InstrumentationScope |

## Notifications

On every span insert or update, the Tracing service publishes an event to the [Notifications](notifications.md) service.

| Event | Room | Published when |
|-------|------|----------------|
| `span.created` | `trace:{trace_id}` | A new span is inserted |
| `span.updated` | `trace:{trace_id}` | An existing span is patched via upsert |

The UI subscribes to `trace:{trace_id}` to receive real-time updates for all spans within a trace being viewed. This enables live visualization of in-progress operations without polling.

## External API

The [Gateway](gateway.md) exposes the Tracing query API via `TracingGateway`:

| Gateway RPC | Internal RPC |
|-------------|-------------|
| `ListSpans` | `TracingService.ListSpans` |
| `GetSpan` | `TracingService.GetSpan` |
| `GetTrace` | `TracingService.GetTrace` |

The ingestion endpoint (`TraceService/Export`) is not proxied through the Gateway. Producers (agents) connect directly to the Tracing service's gRPC endpoint using the standard OTLP exporter configuration.

## Authorization

Access control for tracing data will be handled via ReBAC. The specific relationships and permissions are not yet defined — traces are independent resources not directly associated with an organization. The authorization model will be determined as the broader [ReBAC migration](authz.md) progresses. See [open question: Tracing AuthN/AuthZ](../open-questions.md#tracing-authnz).

## Classification

| Aspect | Detail |
|--------|--------|
| **Plane** | Data |
| **API** | gRPC — OTLP `TraceService/Export` (ingestion) + `TracingService` (query) |
| **State** | PostgreSQL |
| **Scaling** | Scales with ingestion volume and query traffic |
| **Failure impact** | Temporary loss drops incoming spans; existing data remains queryable after recovery |
