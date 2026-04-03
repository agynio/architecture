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

    Impl -->|read/ack messages| Threads
    Impl -->|post responses| Threads
    MCP -->|resolve files| Files
    Impl -->|subscribe| Notifications
    Impl -.->|optional| Tracing
```

| Responsibility | Description |
|---------------|-------------|
| **Read messages** | Pull unacknowledged messages via `GetUnackedMessages` |
| **Acknowledge messages** | Call `AckMessages` after successful processing |
| **Access files via MCP** | File content is accessed on demand by the LLM through the [agyn-files-mcp](../agyn-files-mcp.md) MCP server, which reads from the Files service |
| **Process** | Run implementation-specific logic (LLM calls, tool use, etc.) |
| **Post responses** | Write response messages back to the thread via Threads API |
| **Subscribe to notifications** | Listen for `message.created` events on `thread_participant:{agentId}` room |
| **Use tools via MCP** | Connect to MCP servers for tool access |
| **Report tracing** | Optionally emit tracing data |

The agent is a **pure client** — it connects to the [Gateway](../gateway.md) and the [LLM Proxy](../llm-proxy.md) using OpenZiti service hostnames transparently intercepted by the pod's [Ziti sidecar](../openziti.md#agent-access-scope) and accesses all platform services through them. It does not expose any server or accept inbound connections.

## Communication Protocol

The agent uses a **pull strategy combined with notifications** to receive messages.

### How It Works

```mermaid
sequenceDiagram
    participant N as Notifications
    participant GW as Gateway
    participant A as Agent
    participant T as Threads

    Note over A: Startup
    A->>GW: Subscribe to thread_participant:{agentId} room
    GW->>N: Subscribe (server-streaming)
    A->>GW: GetUnackedMessages(agentId)
    GW->>T: GetUnackedMessages(agentId)
    T-->>GW: Messages
    GW-->>A: Messages
    A->>A: Process messages
    A->>GW: AckMessages
    GW->>T: AckMessages

    Note over A: Mid-run: new message arrives
    N-->>GW: message.created
    GW-->>A: message.created
    A->>GW: GetUnackedMessages(agentId)
    GW->>T: GetUnackedMessages(agentId)
    T-->>GW: Messages
    GW-->>A: Messages
    A->>A: Process new messages
    A->>GW: AckMessages
    GW->>T: AckMessages
```

1. On startup, the agent connects to the [Gateway](../gateway.md) (via the `gateway.ziti` OpenZiti hostname, transparently intercepted by the pod's Ziti sidecar), subscribes to its `thread_participant:{agentId}` notification room and pulls unacknowledged messages via `GetUnackedMessages`. See [Consumer Sync Protocol](../notifications.md#consumer-sync-protocol) for the subscribe/fetch/dedup sequence.
2. During processing, new messages may arrive. The Gateway delivers a `message.created` event (from Notifications), waking the agent to check for new messages at the appropriate point in its processing loop.
3. After processing, the agent calls `AckMessages` to confirm the messages were handled.
4. When idle (current turn complete, no unacknowledged messages), the agent waits for either a notification or the poll interval to expire, then checks again.
5. The polling loop is a **fallback**. The poll interval can be long (10s, 30s) since notifications handle the latency-sensitive path.

### Design Principles

- **Pull at defined loop stages.** The `whenBusy` configuration controls when mid-run messages are picked up: between turns (`wait`) or between tool calls (`injectAfterTools`). The notification wakes the agent, but the actual message read happens at the next check point in the LLM loop.
- **No inbound connections.** The agent connects outbound to the [Gateway](../gateway.md) only (via the `gateway.ziti` OpenZiti hostname, transparently intercepted by the pod's Ziti sidecar). The Gateway routes requests to internal services (Threads, Notifications, Files, etc.). No server, no open port, no service discovery per agent.

## Tools

All tools are provided via **MCP protocol** (Model Context Protocol). The goal is to eliminate built-in tools entirely, making tools reusable across any agent implementation. See [MCP](../mcp.md) for the full MCP architecture.

| Aspect | Details |
|--------|---------|
| Transport | Streamable HTTP (agent CLI → MCP sidecar). stdio MCP servers are wrapped by a sidecar proxy |
| Server location | Sidecar containers in the agent pod (shared network namespace) |
| Connection | Agent CLI connects directly to each MCP server on `localhost:<port>` |
| Namespacing | Owned by each agent CLI implementation |
| Resilience | Health checks + restart with configurable backoff (managed by the sidecar) |

MCP servers are defined as agent resources (see [Agents](../agents-service.md)) and mounted into the agent pod as sidecars by the Runner.

## Wrapper Model

Most 3rd-party agents are implemented as CLIs. The platform provides a **wrapper** that adapts any CLI agent to the platform contract:

```mermaid
sequenceDiagram
    participant GW as Gateway
    participant W as Wrapper
    participant CLI as Agent CLI
    participant MCP as MCP Servers

    W->>GW: Subscribe to thread_participant:{agentId} room
    W->>CLI: Start process with config
    W->>MCP: Start MCP servers
    W->>CLI: Connect MCP servers
    W->>GW: GetUnackedMessages(agentId)
    GW-->>W: Messages
    W->>CLI: Feed messages
    CLI->>MCP: Tool calls
    MCP-->>CLI: Tool results
    CLI-->>W: Output
    W->>GW: Post response to thread
    W->>GW: AckMessages
    Note over W: Wait for notification or poll
    GW-->>W: message.created
    W->>GW: GetUnackedMessages(agentId)
    GW-->>W: Messages
    W->>CLI: Feed new messages
```

The wrapper:
1. Subscribes to notifications for the agent's participant room.
2. Starts the agent CLI process with configuration.
3. Connects MCP tool servers to the agent.
4. Pulls unacknowledged messages (via Gateway → Threads) and feeds them to the CLI.
5. Collects CLI output and posts responses to the thread (via Gateway → Threads).
6. Acknowledges processed messages via `AckMessages`.
7. Waits for notifications or poll fallback for new messages.

## Lifecycle

```mermaid
sequenceDiagram
    participant T as Threads
    participant O as Agents Orchestrator
    participant R as Runner
    participant A as Agent Container

    T->>O: Unacknowledged messages (reconciliation loop)
    O->>R: StartWorkload
    R->>A: Create container
    A->>A: Connect to Gateway, subscribe, pull, process, ack
    A->>A: Post response (via Gateway)

    Note over A,O: Agent idle, waiting for messages
    O->>O: Activity check (reconciliation loop)
    Note over O: Idle timeout exceeded
    O->>R: StopWorkload
```

1. The orchestrator's reconciliation loop detects threads with unacknowledged messages for agent participants.
2. Orchestrator requests Runner to start an agent workload with thread ID and agent config.
3. Runner creates a container with the agent image, MCP sidecars, and configuration.
4. Agent connects to the Gateway (via the `gateway.ziti` OpenZiti hostname, transparently intercepted by the pod's Ziti sidecar), subscribes to notifications, pulls unacknowledged messages, processes, posts responses, acknowledges.
5. Agent waits for new messages (notification or poll fallback).
6. The orchestrator monitors agent activity. When idle timeout is exceeded, it stops the workload via Runner.

### Idle Timeout

The **orchestrator** owns idle timeout enforcement. During each reconciliation pass, it checks running agent workloads against their last activity (last message on the thread). Agents that have been idle beyond the configured timeout are stopped via `Runner.StopWorkload`.

The agent container does not implement idle detection. It may exit naturally (process completion, crash), but the orchestrator is the authority for lifecycle management.

### Scaling

In the simple case, one container per agent invocation. For specific agents, batching may be desirable — a single agent instance processing multiple threads. See [open question](../../open-questions.md#agent-batching-protocol).

## Configuration

The agent resource definition (identity, model, image, compute resources, configuration) is documented in [Resource Definitions](../resource-definitions.md#agent). Sub-resources (MCP servers, volumes, skills, hooks, environment variables, init scripts) that compose the agent's runtime environment are documented alongside the agent in [Resource Definitions](../resource-definitions.md).

Agent implementation-specific behavioral configuration (system prompt, summarization, message buffering) is documented in [Agent Implementation](implementation.md#configuration).
