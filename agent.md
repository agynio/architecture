# Agent Architecture

## Overview

An AI agent is a workload that processes messages from a thread using an LLM. The agent is currently embedded in the platform-server monolith and will be extracted into a standalone container.

## Core Components

```
┌─────────────────────────────────────────────┐
│                   Agent                      │
│                                              │
│  ┌──────────────┐    ┌───────────────────┐  │
│  │   LLM Loop   │◄──►│  Summarization    │  │
│  │              │    │                   │  │
│  │ - Call LLM   │    │ - Rolling summary │  │
│  │ - Route      │    │ - Token budgeting │  │
│  │ - Call tools │    └───────────────────┘  │
│  │ - Send tool  │                            │
│  │   outputs    │    ┌───────────────────┐  │
│  │   back       │    │ State Persistence │  │
│  │ - Manage     │◄──►│                   │  │
│  │   context    │    │ - In-memory       │  │
│  └──────┬───────┘    │ - File-based      │  │
│         │            │ - Remote (APSS)   │  │
│         ▼            └───────────────────┘  │
│  ┌──────────────┐                            │
│  │    Tools     │                            │
│  │              │                            │
│  │ - MCP tools  │                            │
│  │   (primary)  │                            │
│  └──────────────┘                            │
│                                              │
│  Dependencies:                               │
│  ┌──────────────┐    ┌───────────────────┐  │
│  │ LLM Provider │    │ Tracing (optional)│  │
│  └──────────────┘    └───────────────────┘  │
└─────────────────────────────────────────────┘
```

### LLM Loop

The LLM loop is the core execution cycle. It is implemented as a `Loop` of `Reducer` stages connected by `Router` decisions:

1. **Load** — Load conversation context (from state persistence).
2. **Summarize** — If token budget is exceeded, fold older messages into a rolling summary.
3. **Call Model** — Send context to the LLM provider. Prepend system prompt.
4. **Route** — Inspect LLM response:
   - If tool calls are present → go to Call Tools.
   - If restriction enforcement is active and model tried to finish without calling a tool → re-inject restriction and go to Call Model.
   - Otherwise → go to Save.
5. **Call Tools** — Execute each tool call, collect outputs.
6. **Save** — Persist updated conversation state.

Each stage is a `Reducer<State, Context>` that transforms the agent state. Routing between stages is handled by `Router<State, Context>` instances.

### Summarization

Rolling summarization keeps the LLM context within a token budget:

- `summarizationKeepTokens` — Number of most-recent tokens preserved verbatim.
- `summarizationMaxTokens` — Total token budget for the context sent to the LLM.

When the context exceeds the budget, older messages beyond the verbatim tail are folded into a rolling summary by a dedicated LLM call.

### State Persistence

Agent state can be backed by multiple storage strategies:

| Strategy | Description |
|----------|-------------|
| In-memory | Ephemeral, lost on container restart |
| File-based | Persisted to workspace filesystem |
| Remote (APSS) | Agent-State service via gRPC (see [Agent State](agent-state.md)) |

### Tools

Tools extend agent capabilities. The goal is to **eliminate built-in tools and provide all tools via MCP protocol**, making tools reusable across any agent implementation.

Current MCP integration:
- MCP server runs inside a workspace container.
- Communication over stdio using newline-delimited JSON-RPC 2.0.
- Tools are namespace-prefixed (`<namespace>:<toolName>`) to prevent collisions.
- Heartbeat and restart with configurable backoff for resilience.

## Agent Interface

The agent is designed to be implementation-agnostic. Our own agent is the primary implementation, but the interface should support wrapping 3rd-party agents (e.g., any CLI-based agent).

### Wrapper Model

Most 3rd-party agents are implemented as CLIs. The platform provides a wrapper that:
1. Starts the agent CLI process.
2. Provides configuration (model, system prompt, etc.).
3. Connects MCP tool servers to the agent.
4. Collects output and routes it back to the thread.

The communication protocol between the wrapper and the agent is [to be defined](open-questions.md#agent-protocol). If push-based, it will use gRPC. If pull-based, no exposed methods are needed.

## Agent Lifecycle

The **Agents orchestrator** (control plane) is responsible for spinning up agent workloads:

1. Detect threads with pending messages (messages without agent response).
2. Request the Runner to start an agent container with the appropriate configuration.
3. The agent processes messages and posts responses back to the thread.
4. On idle, the agent is stopped and removed.

### Scaling Considerations

In the simple case, one container is created per agent invocation. However, for specific agents, batching may be desirable — a single agent instance processing multiple threads. The batching protocol is an [open question](open-questions.md#agent-batching-protocol).

## Agent Configuration

Defined in the Teams service (see [Teams](teams.md)) as agent resources with the following key fields:

| Field | Description |
|-------|-------------|
| `model` | LLM model identifier (e.g., `gpt-5`) |
| `systemPrompt` | System prompt injected at the start of each turn |
| `debounceMs` | Debounce window for message buffer |
| `whenBusy` | Behavior when agent is busy: `wait` or `injectAfterTools` |
| `processBuffer` | Drain mode: `allTogether` or `oneByOne` |
| `sendFinalResponseToThread` | Auto-send final assistant response to thread |
| `summarizationKeepTokens` | Verbatim token tail size |
| `summarizationMaxTokens` | Summary token budget |
| `restrictOutput` | Enforce tool call before finishing |
| `name` | Agent display name |
| `role` | Agent role label |
