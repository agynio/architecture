# Agent

## Overview

An agent is a workload that processes messages from a thread. The platform is **implementation-agnostic** — our own agent implementation is the primary one, but the interface must support wrapping 3rd-party agents (e.g., Claude Code, Codex CLI, custom CLIs).

This document describes the agent contract: what an agent is, how it connects to the platform, and how its lifecycle is managed. For our specific implementation details, see [Agent Implementation](implementation.md).

## Agent Contract

Every agent, regardless of implementation, must satisfy the same contract:

```mermaid
graph TB
    subgraph "Agent Container"
        Impl[Agent Implementation<br/>our own / 3rd-party CLI / custom]
        MCP[MCP Tool Servers]
        Impl --> MCP
    end

    subgraph Platform
        Threads
        Notifications
        Files
        Tracing
    end

    Impl -->|read messages| Threads
    Impl -->|post responses| Threads
    Impl -->|resolve files| Files
    Impl -->|subscribe| Notifications
    Impl -.->|optional| Tracing
```

| Responsibility | Description |
|---------------|-------------|
| **Read messages** | Pull pending messages from the assigned thread via Threads API |
| **Resolve file URLs** | Request pre-signed download URLs for file references via Files API |
| **Process** | Run implementation-specific logic (LLM calls, tool use, etc.) |
| **Post responses** | Write response messages back to the thread via Threads API |
| **Subscribe to notifications** | Listen for new-message events to react without polling delay |
| **Use tools via MCP** | Connect to MCP servers for tool access |
| **Report tracing** | Optionally emit tracing data |

The agent is a **pure client** — it makes outbound connections to platform services. It does not expose any server or accept inbound connections.

## Communication Protocol

The agent uses a **pull strategy combined with notifications** to receive messages.

### How It Works

```mermaid
sequenceDiagram
    participant N as Notifications
    participant A as Agent
    participant T as Threads

    Note over A: Startup
    A->>N: Subscribe to thread room
    A->>T: GetMessages (unread)
    A->>A: Process messages

    Note over A: Mid-run: new message arrives
    N-->>A: message.created event
    A->>T: GetMessages (unread)
    A->>A: Process new messages

    Note over A: Idle: no messages
    A->>A: Wait for notification or poll interval
    N-->>A: message.created event
    A->>T: GetMessages (unread)
    A->>A: Process messages
```

1. On startup, the agent subscribes to notifications for its thread and pulls pending messages from the Threads API.
2. During processing, new messages may arrive. The Notifications service delivers a `message.created` event instantly, waking the agent to check for new messages at the appropriate point in its processing loop.
3. When idle (current turn complete, no pending messages), the agent waits for either a notification or the poll interval to expire, then checks again.
4. The polling loop is a **fallback** — it catches anything missed during notification gaps (reconnects, race conditions). The poll interval can be long (10s, 30s) since notifications handle the latency-sensitive path.

### Design Principles

- **Notifications are signals, not data.** The notification tells the agent "something happened on your thread." The agent always reads the actual messages from the Threads API. This keeps the Threads API as the single source of truth.
- **Pull at defined loop stages.** The `whenBusy` configuration controls when mid-run messages are picked up: between turns (`wait`) or between tool calls (`injectAfterTools`). The notification wakes the agent, but the actual message read happens at the next check point in the LLM loop.
- **No inbound connections.** The agent connects outbound to Notifications (gRPC subscribe stream), Threads (gRPC calls), and Files (gRPC calls). No server, no open port, no service discovery per agent.

## Tools

All tools are provided via **MCP protocol** (Model Context Protocol). The goal is to eliminate built-in tools entirely, making tools reusable across any agent implementation.

| Aspect | Details |
|--------|---------|
| Transport | stdio (newline-delimited JSON-RPC 2.0) |
| Server location | Inside the workspace container (sidecar) |
| Namespacing | `<namespace>:<toolName>` to prevent collisions |
| Resilience | Heartbeat + restart with configurable backoff |

MCP servers are defined as team resources (see [Teams](../teams.md)) and mounted into the agent container as sidecars by the Runner.

## Wrapper Model

Most 3rd-party agents are implemented as CLIs. The platform provides a **wrapper** that adapts any CLI agent to the platform contract:

```mermaid
sequenceDiagram
    participant W as Wrapper
    participant CLI as Agent CLI
    participant MCP as MCP Servers
    participant T as Threads
    participant N as Notifications

    W->>N: Subscribe to thread room
    W->>CLI: Start process with config
    W->>MCP: Start MCP servers
    W->>CLI: Connect MCP servers
    W->>T: GetMessages (unread)
    W->>CLI: Feed messages
    CLI->>MCP: Tool calls
    MCP-->>CLI: Tool results
    CLI-->>W: Output
    W->>T: Post response to thread
    Note over W: Wait for notification or poll
    N-->>W: message.created
    W->>T: GetMessages (unread)
    W->>CLI: Feed new messages
```

The wrapper:
1. Subscribes to notifications for the assigned thread.
2. Starts the agent CLI process with configuration.
3. Connects MCP tool servers to the agent.
4. Pulls messages from Threads and feeds them to the CLI.
5. Collects CLI output and posts responses to the thread.
6. Waits for notifications or poll fallback for new messages.

## Lifecycle

```mermaid
sequenceDiagram
    participant T as Threads
    participant O as Agents Orchestrator
    participant R as Runner
    participant A as Agent Container

    T->>O: Pending message (reconciliation loop)
    O->>R: StartWorkload
    R->>A: Create container
    A->>A: Subscribe, pull, process
    A->>T: Post response

    Note over A,O: Agent idle, waiting for messages
    O->>O: Activity check (reconciliation loop)
    Note over O: Idle timeout exceeded
    O->>R: StopWorkload
```

1. The orchestrator's reconciliation loop detects threads with pending messages.
2. Orchestrator requests Runner to start an agent workload with thread ID and agent config.
3. Runner creates a container with the agent image, MCP sidecars, and configuration.
4. Agent subscribes to notifications, pulls messages, processes, posts responses.
5. Agent waits for new messages (notification or poll fallback).
6. The orchestrator monitors agent activity. When idle timeout is exceeded, it stops the workload via Runner.

### Idle Timeout

The **orchestrator** owns idle timeout enforcement. During each reconciliation pass, it checks running agent workloads against their last activity (last message on the thread). Agents that have been idle beyond the configured timeout are stopped via `Runner.StopWorkload`.

The agent container does not implement idle detection. It may exit naturally (process completion, crash), but the orchestrator is the authority for lifecycle management.

### Scaling

In the simple case, one container per agent invocation. For specific agents, batching may be desirable — a single agent instance processing multiple threads. See [open question](../../open-questions.md#agent-batching-protocol).

## Configuration

Agent configuration is defined in the Teams service as agent resources:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Agent display name |
| `role` | string | Agent role label |
| `model` | string | LLM model identifier (e.g., `gpt-5`) |
| `systemPrompt` | string | System prompt injected at start of each turn |
| `debounceMs` | integer | Debounce window for message buffer (ms) |
| `whenBusy` | enum | `wait` or `injectAfterTools` |
| `processBuffer` | enum | `allTogether` or `oneByOne` |
| `sendFinalResponseToThread` | boolean | Auto-send final response to thread |
| `restrictOutput` | boolean | Enforce tool call before finishing |

Implementation-specific configuration (e.g., summarization parameters) is documented in [Agent Implementation](implementation.md#configuration).
