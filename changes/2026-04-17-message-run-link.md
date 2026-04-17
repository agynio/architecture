# Message–Run Link

## Target

- [Threads — Data Model](../architecture/threads.md#data-model)
- [Tracing — agynd Tracing Proxy](../architecture/tracing.md#agynd-tracing-proxy)
- [Tracing — Attribute Injection and Verification](../architecture/tracing.md#attribute-injection-and-verification)
- [Tracing — Query API](../architecture/tracing.md#query-api)
- [Tracing — Indexes](../architecture/tracing.md#indexes)
- [Agents Orchestrator — Workload Spec Assembly](../architecture/agents-orchestrator.md#workload-spec-assembly)
- [Chat — Product](../product/chat/chat.md)
- [Run Timeline — Product](../product/tracing/run-timeline.md)

## Problem

Chat messages and traces have no connection. Spans carry `agyn.thread.id` (thread-level scope) but there is no way to navigate from a specific chat message to the run that processed it.

## Solution

Store `agyn.thread.message.id` and `agyn.workload.id` as resource attributes on spans, injected by the `agynd` tracing proxy. The chat UI opens the tracing app with the `message_id`. The tracing app queries spans by `agyn.thread.message.id`, resolves the `agyn.workload.id` from the result, and displays the corresponding run.

## Delta

### agynd tracing proxy: inject `message_id` and `workload_id`

`agynd` tracks the ID of the message currently being processed (the message it most recently fed to the agent CLI for the current turn). On each `Export` request, the proxy injects two additional resource attributes alongside the existing `agyn.thread.id`:

| Resource Attribute | Source | Description |
|--------------------|--------|-------------|
| `agyn.thread.message.id` | current message being processed | UUID of the thread message that triggered this agent turn |
| `agyn.workload.id` | `WORKLOAD_ID` environment variable | Workload UUID for this execution |

`agyn.thread.message.id` is a mutable proxy-side value — `agynd` updates it each time it feeds a new message to the agent CLI. All span exports that arrive at the proxy while the agent is processing a given message carry that message's ID.

### Agents Orchestrator: expose `WORKLOAD_ID`

The orchestrator adds `WORKLOAD_ID` to the platform-managed environment variables injected into the agent container:

| Variable | Injected into | Description |
|----------|---------------|-------------|
| `WORKLOAD_ID` | Agent container | Workload UUID (`workload_key`) for this execution |

### Tracing service: verify, index, and expose `agyn.thread.message.id` and `agyn.workload.id`

**Attribute verification** — `agyn.thread.message.id` is a producer-asserted attribute (set by `agynd`, not derived from connection identity). The Tracing service verifies it against the Threads service: the message must belong to the thread identified by `agyn.thread.id`. If verification fails, the entire `Export` request is rejected.

`agyn.workload.id` is passed through as-is — no verification required beyond storage.

**Index** — add a partial index on the `agyn.thread.message.id` resource attribute to support efficient lookups by message.

**SpanFilter** — add `message_id` filter field to `ListSpans`:

| Field | Type | Description |
|-------|------|-------------|
| `message_id` | string (UUID), optional | Return only spans attributed to this thread message |

### Chat UI: "View trace" link per message

For every message in the conversation area, show a **"View trace"** link. Clicking it opens the Run Timeline with the `message_id`. The tracing app receives the `message_id`, resolves the run, and displays it. The chat UI passes the parameter and does nothing else.

### Run Timeline: message_id entry point

The Run Timeline accepts an optional `message_id` query parameter in addition to the existing direct `runId` URL path.

When `message_id` is present:

1. Call `ListSpans(filter: {thread_id, message_id})`.
2. Extract `agyn.workload.id` from the returned spans — this is the `run_id`.
3. Load and display the Run Timeline for that run, with the message-attributed spans highlighted.

If no spans are found for the given `message_id` (tracing data expired or agent produced no spans), display an empty state with an explanatory message.

## Acceptance Signal

- Spans exported during processing of message M carry `agyn.thread.message.id = M` and `agyn.workload.id = W`.
- `ListSpans(filter: {message_id: M})` returns only spans attributed to message M.
- Clicking "View trace" on message M in the chat UI opens the Run Timeline for workload W, with M's spans highlighted.
- Spans with an invalid `agyn.thread.message.id` (message not in the attributed thread) are rejected by the Tracing service.

## Notes

- No change to the Message data model — the link lives entirely in tracing data.
- A single agent turn maps to one message ID. If an agent processes multiple messages before responding, each message gets its own set of attributed spans.
- Tracing has shorter retention than Threads. If trace data has expired, "View trace" shows an empty state rather than breaking the chat UI.
