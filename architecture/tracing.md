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
| `(agyn.thread.message.id)` partial | Look up spans by originating message |

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
| `message_id` | string (UUID), optional | Return only spans attributed to this thread message (`agyn.thread.message.id`) |

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

The ingestion endpoint (`TraceService/Export`) is not proxied through the Gateway. [Producers](#span-producers) connect to the Tracing service via its own [OpenZiti service](#ingestion-authentication) (`tracing.agyn`), each authenticated by the identity it holds.

## Authentication and Authorization

### Ingestion Authentication

The Tracing service participates in the OpenZiti overlay. It obtains its identity at runtime via [self-enrollment](openziti.md#service-identity-self-enrollment) through [Ziti Management](openziti.md#ziti-management-service), the same pattern as the Gateway and LLM Proxy.

| Aspect | Detail |
|--------|--------|
| Role attributes | `["tracing-hosts"]` |
| Service name | `tracing` |
| Enrollment | Self-enrollment via Ziti Management at pod startup |
| SDK usage | `zitiContext.ListenWithOptions("tracing", ...)` — binds the `tracing` service |

Agents connect to the Tracing service via the `tracing.agyn` OpenZiti hostname, transparently intercepted by the pod's Ziti sidecar. Authentication is mTLS — the Tracing service extracts the caller's OpenZiti identity from the connection via `conn.GetDialerIdentityId()` and resolves it to a platform identity via [Ziti Management](openziti.md) `ResolveIdentity`, the same mechanism as the [Gateway](gateway.md) and [LLM Proxy](llm-proxy.md).

Any authenticated agent can export spans. There is no authorization check on the ingestion path — if the agent has a valid OpenZiti identity, it can export. Per-span attribute verification is described in [Attribute Injection and Verification](#attribute-injection-and-verification).

#### Static Policies

Two new static policies at bootstrap:

| Policy | Type | Identity Roles | Service Roles | Purpose |
|--------|------|---------------|---------------|---------|
| `agents-dial-tracing` | Dial | `#agents` | `@tracing` | Agents can reach Tracing service |
| `tracing-bind` | Bind | `#tracing-hosts` | `@tracing` | Tracing service hosts the `tracing` service |

### Query Authorization

The query API is proxied through the [Gateway](gateway.md) via `TracingGateway`. The Gateway authenticates the caller (OIDC, API token, or OpenZiti). Query results are scoped by organization — a caller must be a member of an organization to view traces attributed to that organization:

```
Check(identity:<callerId>, member, organization:<orgId>) → allowed: bool
```

The `organization_id` filter is a required parameter on `ListSpans`. `GetTrace` and `GetSpan` check the caller's membership in the organization associated with the returned trace data.

### Attribute Injection and Verification

Producers do not set identity-based resource attributes. The Tracing service derives and injects those from the authenticated connection, and verifies the few a producer may assert (`agyn.agent_instance.id`, `agyn.thread.message.id`).

#### Injected by the Tracing service (from connection identity)

On each `Export` request, the Tracing service resolves the caller's identity chain and **overwrites** the following resource attributes on every `ResourceSpans` in the request:

| Resource Attribute | Source | Description |
|--------------------|--------|-------------|
| `agyn.identity.id` | OpenZiti mTLS → `ZitiManagement.ResolveIdentity` | Platform identity UUID (the agent instance's identity) |
| `agyn.agent.id` | `Agents.ResolveAgentIdentity(identity_id)` | Agent class UUID (resolved through the instance) |
| `agyn.organization.id` | `Agents.ResolveAgentIdentity(identity_id)` | Organization UUID |

These attributes are never trusted from the producer. The Tracing service always overwrites them with values derived from the verified network identity. This prevents a compromised agent pod from misattributing spans to a different agent or organization.

#### Verified by the Tracing service (from producer)

A producer may assert the following resource attributes. The Tracing service verifies each one:

**`agyn.agent_instance.id`** — must equal the caller's verified identity (`identity_id == agent_instance_id` by construction). Mismatch rejects the `Export` request. A workload authenticates as its instance, so it need not assert this at all; the service derives the same value from the connection.

**`agyn.thread.message.id`** — asserted only by [`agynd`](agynd-cli.md), on the `invocation.message` it emits for the item it fed the agent CLI. Verified against the [Threads](threads.md) service: the message must belong to a thread the caller `can_read`, checked against [Authorization](authz.md). A failed check rejects the entire `Export` request.

No producer asserts a thread. An [agent instance](agent-instances.md) serves an inbox drawn from many threads, so a span belongs to no one thread — only to the message that opened the turn it sits under, which the message attribute carries and the thread is resolved from.

`agyn.workload.id` is stored as-is without verification. It names the wake cycle, and every producer in the container reads it from the same environment.

#### Resolution Caching

The identity chain resolution (`identity_id → agent_id, organization_id`) and thread authorization checks are cached in an LRU cache.

| Aspect | Detail |
|--------|--------|
| Cache type | LRU |
| Cache keys | `identity_id` (for identity chain), `(identity_id, thread_id)` (for thread authorization) |
| Max entries | Configurable, default `1000` |
| Invalidation | TTL-based. Agent pod identities are ephemeral — entries expire naturally |

### Agents Service: ResolveAgentIdentity

The [Agents](agents-service.md) service provides a new method for resolving an agent's identity to its resource metadata:

| Method | Description |
|--------|-------------|
| `ResolveAgentIdentity` | Given an `identity_id`, return the agent's resource ID and organization ID |

**Request:**

| Field | Type | Description |
|-------|------|-------------|
| `identity_id` | string (UUID) | Platform identity UUID |

**Response:**

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | string (UUID) | Agent resource UUID |
| `organization_id` | string (UUID) | Organization the agent belongs to |

Returns `NOT_FOUND` if the identity does not correspond to an agent. This method is internal — not exposed through the Gateway.

## Egress Spans

The [Egress Gateway](egress-gateway.md) emits one span per outbound request it handles. Spans follow the standard OTel format; the attributes below identify the egress action.

| Span field | Value |
|---|---|
| `name` | `egress.request` |
| `kind` | `CLIENT` |
| `status.code` | `OK` for `allow`/`deny` outcomes (the gateway acted as configured); `ERROR` for `upstream_error` and `tls_error` |

Attributes:

| Attribute | Type | Description |
|---|---|---|
| `egress.method` | string | HTTP method (`GET`, `POST`, …) |
| `egress.host` | string | Destination host from SNI (HTTPS) or `Host` header (HTTP) |
| `egress.port` | int | Destination port |
| `egress.path` | string | Request path with query string stripped |
| `egress.outcome` | enum | `allow`, `deny`, `upstream_error`, `tls_error` |
| `egress.matched_rule_ids` | string | Comma-separated UUIDs of all rules whose `effect.action` or `effect.inject` was applied |
| `egress.upstream_status` | int | HTTP status returned by the upstream (when reached) |
| `egress.bytes_in` | int | Request body bytes forwarded |
| `egress.bytes_out` | int | Response body bytes forwarded |
| `agyn.agent.id` | string (UUID) | Agent the request originated from (derived from the OpenZiti identity) |
| `agyn.workload.id` | string (UUID) | Workload that issued the request |
| `agyn.organization.id` | string (UUID) | Organization owning the agent |

Header values, request bodies, and response bodies are **not** recorded — the span is sufficient to answer "which agent called which destination when, and was it allowed."

The gateway sets `agyn.agent.id`, `agyn.workload.id`, and `agyn.organization.id` itself (it has them from the verified OpenZiti identity). It does not set `agyn.thread.message.id` — egress requests are not scoped to a single thread message.

## Span Producers

Spans reach the Tracing service directly from whatever produced them. Each producer holds an OpenZiti identity and dials `tracing.agyn` itself; the service attributes what arrives to the identity that sent it.

| Producer | Emits |
|----------|-------|
| [`agynd`](agynd-cli.md) | `invocation.message`, when it feeds an inbox item to the agent CLI. It also opens the trace the others write into |
| [Trace hook](#the-trace-hook) | `llm.call` and `tool.execution`, reconstructed from the agent CLI's session transcript — for the CLIs that keep one |
| [The agent CLI itself](#a-cli-that-exports-its-own-spans) | the same, exported directly, when the CLI is one the platform builds and can therefore be told where to send them |
| Platform services | spans for the work a request passes through |

Two of these produce the same spans by different means, and which applies is a
property of the CLI rather than a choice. A third-party CLI exports what its
author decided to export, so the platform reads its transcript instead; a CLI
the platform builds emits the shape this service wants directly, and reading it
back out of a file would be a detour. Both write into the trace `agynd` opened,
and both are attributed the same way — the pod's Ziti sidecar carries the
connection and the workload's identity authenticates it.

### The trace is the wake cycle

A workload is started when an [agent instance](agent-instances.md) has work and stopped when it goes idle, so the workload *is* one wake cycle. `agynd` opens a trace for it during [environment preparation](agynd-cli.md#3-environment-preparation) and hands the id to the trace hook it installs, so both producers write into one trace.

The id is derived from `WORKLOAD_ID` rather than drawn, so an `agynd` restarting inside the pod reopens the trace it was already writing instead of splitting the cycle in two.

A cycle serves an inbox that spans threads, so its trace holds many turns. Each `invocation.message` marks where one begins, which is what lets a reader show a single turn or the whole cycle.

### Attribution

The Tracing service derives attribution from the verified OpenZiti connection rather than trusting the producer to assert it. A workload authenticates as its agent instance, which is what `agyn.agent_instance.id` means, so the identity *is* the attribution.

No producer asserts a thread. An instance serves an inbox drawn from many threads, so there is no thread a span belongs to — only the message that opened the turn it sits under, which `invocation.message` carries.

## The Trace Hook

What an agent did is read from the **session transcript** the agent CLI writes, not from the telemetry it exports. `agynd` registers a hook on each CLI's turn completion; the hook reads that transcript and exports the reconstructed turn to the Tracing service.

### Why not a third-party CLI's own telemetry

A third-party agent CLI's OTel export describes that a model call happened — endpoint, model, status, duration, token counts — and never what was said. Prompts appear as a length, streamed output as a chunk count, tool calls not at all. The span tree it produces describes the framework's internals: one span per streamed chunk, nested as deep as the call stack, answering no question the run view asks.

The transcript is the opposite. It is the record the CLI keeps in order to resume a session, so it holds the prompt, the assistant's reply, every tool call with its input and output, and the usage counts — the material this service exists to store.

None of this is an argument against a CLI exporting spans, only against
exporting *those* spans. A CLI the platform builds emits the shape described
here directly — see [A CLI that exports its own spans](#a-cli-that-exports-its-own-spans).

### Hook contract

The hook is a single platform binary, shipped with [`agynd`](agynd-cli.md) and delivered by the same init container, so one implementation serves every CLI: a hook is a command to run, and what differs between CLIs is only the transcript it is handed. `agynd` registers it during [environment preparation](agynd-cli.md#3-environment-preparation), telling it which format to expect and which trace to write into. On each invocation it:

1. Reads the session transcript the CLI identifies.
2. Reconstructs the turns not yet sent, in the shape described below.
3. Exports them as spans to the Tracing service at `tracing.agyn`, in the trace `agynd` opened. The pod's Ziti sidecar carries the connection and the workload's identity authenticates it, so the hook holds no credential of its own.
4. Records what it sent, so a resumed session — which replays the transcript from its start — uploads each turn once.
5. Swallows its own failures. Tracing is an optional dependency; the hook never ends a turn that otherwise succeeded.

| Agent CLI | Hook | Transcript |
|-----------|------|------------|
| **Codex** | `Stop`, registered in `config.toml`; the rollout path arrives on stdin | rollout JSONL |
| **Claude Code** | `Stop` and `SessionEnd`, registered in `~/.claude/settings.json` | conversation transcript |

[agn](agn-cli.md) is absent deliberately — see below.

### A CLI that exports its own spans

`agn` is the platform's own CLI, so the argument for reading a transcript does
not apply to it: nothing has to be reconstructed from a file when the process
that did the work can describe it as it happens. It exports OTLP directly to the
Tracing service, which is an OTLP collector, and emits the same `llm.call` and
`tool.execution` spans the hook reconstructs for the others.

`agynd` tells it where to send them. The orchestrator stamps
`OTEL_EXPORTER_OTLP_ENDPOINT` on the workload alongside `TRACING_ADDRESS`, both
naming `tracing.agyn`, and `agn` picks it up as any OTel process would. An
endpoint that names nothing listening is worse than none at all: the exporter
blocks the turn it is describing until it times out, and the turn's own work is
delayed by a dependency the platform calls optional. Absent, `agn` runs with no
exporter and simply produces no spans.

What `agn` cannot derive on its own is the trace to write into. That is the wake
cycle's, derived from `WORKLOAD_ID`, and it reaches `agn` the same way it reaches
the hook — handed over by `agynd` at start.

### Turn shape

A turn becomes an `llm.call` per model step — carrying the model, the full request context and the usage counts — and a `tool.execution` per tool call, carrying its input and output. Each execution hangs off the call that invoked it, and a subagent's turns hang off the turn that spawned them.

Calls root in the trace rather than under an [`invocation.message`](#span-producers). A transcript names a turn in the CLI's own terms — a line id, a session and a timestamp — never the platform's message id, so the hook has no message span to point at. The shared trace is what joins them, and `invocation.message` marks where each turn begins.

Span ids are derived from the turn rather than drawn, so a resumed session that replays the transcript writes the ids it wrote before and the [upsert](#upsert-semantics) lands on the existing rows. The same derivation completes a turn first exported while it was still running.

Spans are timestamped from the transcript, so a turn is recorded with the times at which it happened rather than the time it was uploaded.

## Configuration

| Field | Source | Description |
|-------|--------|-------------|
| `ZITI_MANAGEMENT_ADDRESS` | Deployment config | gRPC address of the [Ziti Management](openziti.md) service |
| `AGENTS_SERVICE_ADDRESS` | Deployment config | gRPC address of the [Agents](agents-service.md) service |
| `AUTHORIZATION_SERVICE_ADDRESS` | Deployment config | gRPC address of the [Authorization](authz.md) service |
| `IDENTITY_RESOLUTION_CACHE_SIZE` | Deployment config | Max LRU cache entries for identity chain resolution (default: `1000`) |
| `THREAD_AUTH_CACHE_SIZE` | Deployment config | Max LRU cache entries for thread authorization checks (default: `1000`) |

## Classification

| Aspect | Detail |
|--------|--------|
| **Plane** | Data |
| **API** | gRPC — OTLP `TraceService/Export` (ingestion via OpenZiti) + `TracingService` (query) |
| **State** | PostgreSQL |
| **Scaling** | Scales with ingestion volume and query traffic |
| **Failure impact** | Temporary loss drops incoming spans; existing data remains queryable after recovery |
