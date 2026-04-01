# Run Timeline

## Purpose

Run Timeline is the observability interface for a single agent run. It shows every event in an agent's execution — messages received, LLM calls made, tools executed, and context summarizations — in chronological order with full detail. Operators and developers use it to understand what an agent did, why it made specific decisions, and where failures occurred.

## User Stories

- As an operator, I want to see all events in a run in chronological order so I can reconstruct the agent's execution.
- As an operator, I want to filter events by type (message, LLM, tool, summary) so I can focus on a specific aspect of execution.
- As an operator, I want to filter events by status (running, finished, failed, terminated) so I can isolate problems.
- As an operator, I want to see token usage breakdowns so I can understand cost and context consumption.
- As an operator, I want to follow new events in real time while a run is active so I can monitor live execution.
- As an operator, I want to inspect LLM context (the full prompt sent to the model) so I can understand the input that produced a given response.
- As an operator, I want to see tool execution output (stdout/stderr) streamed in real time so I can watch long-running tools.
- As an operator, I want to terminate a running execution so I can stop a misbehaving agent.
- As a developer, I want keyboard shortcuts for fast navigation so I can inspect events efficiently.

## Behavior

### Layout

The screen is divided into three vertical sections:

1. **Run Header** — top bar showing run status, duration, creation timestamp, and aggregate token usage.
2. **Event List** — left sidebar listing events with type icons, labels, timestamps, and status indicators.
3. **Event Detail** — main content area showing the selected event's full data.

### Run Header

Displays:

- **Status indicator** — visual indicator (running, finished, failed, terminated) with label.
- **Duration** — elapsed time since run start. For running executions, this is a live counter.
- **Creation timestamp** — formatted as "MMM D, YYYY, HH:MM AM/PM".
- **Token usage** — total token count, with a hover popover showing the breakdown:
  - Input tokens.
  - Cached input tokens.
  - Output tokens.
  - Reasoning tokens.
  - Total.
- **Terminate button** — visible only when status is "running". Terminates the run.

### Event List

Lists all events in the run, ordered chronologically (oldest at top, newest at bottom). On initial load, the list scrolls to the bottom (most recent event).

Each event entry shows:

- **Type icon** — color-coded by event kind:
  - Message (blue) — further labeled by subtype: Source, Intermediate, or Result.
  - LLM Call (purple).
  - Tool (cyan) — icon varies: terminal for shell tools, users for management tools, wrench for others.
  - Summarization (gray).
- **Label** — event name (e.g., "LLM Call", tool name, "Message • Source").
- **Status indicator** — running, finished, failed, terminated.
- **Timestamp** — formatted time.
- **Duration** — if the event has a known duration.

The active selection is highlighted with a blue left-border indicator.

**Statistics popover** — the event count in the header is hoverable, showing:
- Breakdown by type (message, LLM, tool, summary counts).
- Breakdown by status (running, finished, failed, terminated counts).

**Pagination** — events load with cursor-based pagination. Scrolling to the top triggers "load more" for older events.

**Filtering** — a settings dropdown allows toggling visibility of event types and statuses independently. Filters are additive (selecting "LLM" + "Tool" shows both). When no filters are active, all events are shown.

### Follow Mode

A toggle button enables "follow new events" mode. When active:

- New events appearing at the bottom are automatically selected.
- The list auto-scrolls to keep the latest event visible.

Keyboard shortcut: `F` toggles follow mode.

### Event Detail — Message

Displays:

- Role and kind labels.
- Message text rendered as markdown.
- Raw data in a collapsible JSON viewer.

### Event Detail — LLM Call

Displays:

- **Model** — provider and model name.
- **Parameters** — temperature, topP.
- **Stop reason** — why the LLM stopped generating.
- **Token usage** — input, cached, output, reasoning, total for this specific call.
- **Reasoning metrics** — reasoning tokens and any reasoning effort score extracted from metadata.
- **Response** — the LLM's text response rendered as markdown.
- **Tool calls** — if the LLM requested tool executions, each is listed with name, arguments (JSON), and a link to the corresponding tool execution event.
- **Context (full prompt)** — a paginated, scrollable list of context items (system prompts, user messages, tool results, memory, summaries). Each item shows its role, size, and content. New items (added since the last LLM call) are highlighted. Context loads page-by-page from most recent to oldest.
  - Context delta status indicators: "available" (context shown), "first_call" (first LLM call — all context is new), "empty" (no context), "unavailable" / "redacted" (not accessible).
- **Raw response** — collapsible JSON viewer showing the full raw response.

### Event Detail — Tool Execution

Displays:

- **Tool name** and call ID.
- **Execution status** — success or error.
- **Input** — the arguments passed to the tool, shown as JSON.
- **Output** — the tool's return value, shown as JSON.
- **Terminal output** — for tools that produce stdout/stderr:
  - Streams output in real time via WebSocket while the tool is running.
  - Shows terminal metadata: exit code, bytes written (stdout/stderr), total chunks, dropped chunks, saved path, terminal status (success, error, timeout, idle_timeout, cancelled, truncated).
- **Error details** — error message if the execution failed.
- **Raw data** — collapsible JSON viewer.

### Event Detail — Summarization

Displays:

- **Summary text** — rendered as markdown.
- **Metrics** — new context count and old context token count.
- **Raw data** — collapsible JSON viewer.

### Keyboard Navigation

- **Arrow Up / Arrow Down** — move selection through the event list.
- **F** — toggle follow mode.

### Real-Time Updates

While a run is active, the view receives updates via WebSocket:

- New events appear in the list.
- Existing events update (e.g., pending → running → success).
- Run summary (status, counts, token totals) refreshes on each event update.
- Tool output streams in real time.
- When the run completes or is terminated, the status updates and the live duration counter stops.

## Constraints

- Run timeline is read-only except for the terminate action.
- Context items can be large; pagination prevents loading the entire context at once.
- Tool output streaming may drop chunks under high throughput (dropped chunk count is displayed).
- The view is accessed via a direct URL: `/agents/threads/{threadId}/runs/{runId}/timeline`.
