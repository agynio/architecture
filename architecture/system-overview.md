# System Overview

Agyn is a Kubernetes-native AI agent orchestrator. It manages the lifecycle of AI agents that communicate with humans and each other through threaded conversations, with tools provided via MCP (Model Context Protocol).

## Component Diagram

```mermaid
graph TB
    subgraph External
        WebApp[Web App]
        MobileApp[Mobile App]
        ThirdParty[3rd-party Apps<br/>Slack, etc.]
    end

    subgraph Control Plane
        Teams[Teams]
        AgentsOrch[Agents<br/>Orchestrator]
    end

    subgraph Data Plane
        Gateway[Gateway]
        Channels[Channels]
        Threads[Threads]
        Notifications[Notifications]
        Runner[Runner]
        AgentState[Agent State]
        Tracing[Tracing]
    end

    subgraph Workloads
        Agent1[Agent Container]
        Agent2[Agent Container]
        MCP1[MCP Server]
        MCP2[MCP Server]
    end

    WebApp & MobileApp --> Gateway
    ThirdParty <--> Channels
    WebApp & MobileApp --> Channels

    Gateway --> Threads
    Gateway --> Notifications
    Channels <--> Threads
    Channels --> Notifications

    Threads --> Notifications
    AgentsOrch --> Runner
    AgentsOrch --> Threads

    Runner --> Agent1 & Agent2
    Agent1 --> MCP1
    Agent2 --> MCP2
    Agent1 & Agent2 --> AgentState
    Agent1 & Agent2 -.-> Tracing

    Teams --> AgentsOrch
```

## Component Summary

| Component | Responsibility |
|-----------|---------------|
| **Channels** | Bidirectional interface connecting 3rd-party products (Slack, etc.) and own apps (web, mobile) with Threads |
| **Threads** | Conversation messaging between multiple participants (humans and agents) |
| **Notifications** | Real-time event fanout via persistent connections (socket). Delivers events to relevant clients |
| **Agents** | Orchestrator that spins up agent workloads for threads with pending messages |
| **Agent State** | Long-term agent context persistence (APSS) |
| **Tracing** | Ingestion and query of tracing data. Extended OpenTelemetry protocol for real-time in-progress events |
| **Teams** | Management of team resources: agents, workspaces, MCP servers, etc. |
| **Runner** | Executes workloads. Implementations: docker-runner, k8s-runner |
| **Gateway** | Exposes platform methods for external usage |

## Data Stores

| Store | Current Usage |
|-------|--------------|
| PostgreSQL | Primary relational store (agent state, platform data) |
| Redis | Pub/sub for notifications, caching |
| Filesystem | Graph store (agent graph definitions persisted as filesystem dataset) |

## Repository Map

| Repository | Contents | Language | Status |
|------------|----------|----------|--------|
| `agynio/api` | API schemas: protobuf (internal gRPC) and OpenAPI (external) | Proto, YAML | Active |
| `agynio/platform` | Monolith: platform-server, docker-runner, LLM package, platform-ui | TypeScript | Active (being decomposed) |
| `agynio/notifications` | Notifications service | Go | Standalone service |
| `agynio/gateway` | Gateway service | Go | Standalone service |
| `agynio/agent-state` | Agent State (APSS) service | Go | Standalone service |
| `agynio/architecture` | This documentation | Markdown | — |
