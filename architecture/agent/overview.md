# Agent

## Overview

An agent is a workload that processes messages from a thread. The platform is **implementation-agnostic** — our own agent implementation is the primary one, but the interface must support wrapping 3rd-party agents (e.g., Claude Code, Codex CLI, custom CLIs).

This document describes the agent contract: what an agent is, how it connects to the platform, and how its lifecycle is managed. For our specific implementation details, see [Agent Implementation](implementation.md).

## Agent Contract

Every agent, regardless of implementation, must satisfy the same contract. The agent workload structure is defined in [Orchestrator — Agent Workload](../orchestrator.md#agent-workload).

```mermaid
graph TB
    subgraph "Agent Workload (pod)"
        Impl[Agent Implementation]
        MCP1[MCP Sidecar 1]
        MCP2[MCP Sidecar 2]
        Ziti[OpenZiti Tunnel]
        Impl -->|gRPC| MCP1
        Impl -->|gRPC| MCP2
    end

    subgraph Platform
        Threads
        Notifications
        Files
        Tracing
    end

    Impl -->|read/ack messages| Threads
    Impl -->|post responses| Threads
    Impl -->|resolve files| Files
    Impl -->|subscribe| Notifications
    Impl -.->|optional| Tracing
```

| Responsibility | Description |
|---------------|-------------|
| **Read messages** | Pull unacknowledged messages via `GetUnackedMessages` |
| **Acknowledge messages** | Call `AckMessages` after successful processing |
| **Resolve file URLs** | Request pre-signed download URLs for file references via Files API |
| **Process** | Run implementation-specific logic (LLM calls, tool use, etc.) |
| **Post responses** | Write response messages back to the thread via Threads API |
| **Subscribe to notifications** | Listen for `message.created` events on `thread_participant:{agentId}` room |
| **Use tools via MCP** | Call [MCP sidecars](../mcp-adapter.md) via gRPC |
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
    A->>N: Subscribe to thread_participant:{agentId} room
    A->>T: GetUnackedMessages(agentId)
    A->>A: Process messages
    A->>T: AckMessages

    Note over A: Mid-run: new message arrives
    N-->>A: message.created
    A->>T: GetUnackedMessages(agentId)
    A->>A: Process new messages
    A->>T: AckMessages

    Note over A: Idle: no messages
    A->>A: Wait for notification or poll interval
    N-->>A: message.created
    A->>T: GetUnackedMessages(agentId)
    A->>A: Process messages
    A->>T: AckMessages
```

1. On startup, the agent subscribes to its `thread_participant:{agentId}` notification room and pulls unacknowledged messages from Threads via `GetUnackedMessages`. See [Consumer Sync Protocol](../notifications.md#consumer-sync-protocol) for the subscribe/fetch/dedup sequence.
2. During processing, new messages may arrive. The Notifications service delivers a `message.created` event, waking the agent to check for new messages at the appropriate point in its processing loop.
3. After processing, the agent calls `AckMessages` to confirm the messages were handled.
4. When idle (current turn complete, no unacknowledged messages), the agent waits for either a notification or the poll interval to expire, then checks again.
5. The polling loop is a **fallback**. The poll interval can be long (10s, 30s) since notifications handle the latency-sensitive path.

### Design Principles

- **Pull at defined loop stages.** The `whenBusy` configuration controls when mid-run messages are picked up: between turns (`wait`) or between tool calls (`injectAfterTools`). The notification wakes the agent, but the actual message read happens at the next check point in the LLM loop.
- **No inbound connections.** The agent connects outbound to Notifications (gRPC subscribe stream), Threads (gRPC calls), and Files (gRPC calls). No server, no open port, no service discovery per agent.

## Tools

All tools are provided via **MCP protocol** (Model Context Protocol) through [MCP Adapter](../mcp-adapter.md) sidecars. The goal is to eliminate built-in tools entirely, making tools reusable across any agent implementation.

| Aspect | Details |
|--------|---------|
| Transport | gRPC (agent → adapter sidecar) |
| Protocol bridge | [MCP Adapter](../mcp-adapter.md) bridges gRPC ↔ stdio or Streamable HTTP |
| Namespacing | `<namespace>:<toolName>` to prevent collisions |
| Resilience | Adapter handles heartbeat + restart with configurable backoff |

MCP servers are defined as team resources (see [Teams](../teams.md)) and attached to agents via [attachments](../resource-definitions.md#attachment). The [Agents Orchestrator](../orchestrator.md) includes them as sidecars when assembling the workload.

## Wrapper Model

Most 3rd-party agents are implemented as CLIs. The platform provides a **wrapper** that adapts any CLI agent to the platform contract:

```mermaid
sequenceDiagram
    participant W as Wrapper
    participant CLI as Agent CLI
    participant MCP as MCP Sidecars
    participant T as Threads
    participant N as Notifications

    W->>N: Subscribe to thread_participant:{agentId} room
    W->>CLI: Start process with config
    W->>MCP: Connect via gRPC
    W->>CLI: Connect MCP servers
    W->>T: GetUnackedMessages(agentId)
    W->>CLI: Feed messages
    CLI->>MCP: Tool calls
    MCP-->>CLI: Tool results
    CLI-->>W: Output
    W->>T: Post response to thread
    W->>T: AckMessages
    Note over W: Wait for notification or poll
    N-->>W: message.created
    W->>T: GetUnackedMessages(agentId)
    W->>CLI: Feed new messages
```

The wrapper:
1. Subscribes to notifications for the agent's participant room.
2. Starts the agent CLI process with configuration.
3. Connects to MCP sidecars via gRPC.
4. Pulls unacknowledged messages from Threads and feeds them to the CLI.
5. Collects CLI output and posts responses to the thread.
6. Acknowledges processed messages via `AckMessages`.
7. Waits for notifications or poll fallback for new messages.

## Lifecycle

The [Agents Orchestrator](../orchestrator.md) manages the full workload lifecycle — starting pods when demand exists, stopping them when idle.

```mermaid
sequenceDiagram
    participant T as Threads
    participant O as Agents Orchestrator
    participant R as Runner
    participant A as Agent Workload

    T->>O: Unacknowledged messages (reconciliation loop)
    O->>R: StartWorkload
    R->>A: Create pod
    Note over A: Sidecars start, agent starts
    A->>A: Subscribe, pull, process, ack
    A->>T: Post response

    Note over A,O: Agent idle, waiting for messages
    O->>O: Activity check (reconciliation loop)
    Note over O: Idle timeout exceeded
    O->>R: StopWorkload
```

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
