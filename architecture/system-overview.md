# System Overview

Agyn is a Kubernetes-native AI agent orchestrator. It manages the lifecycle of AI agents that communicate with humans and each other through threaded conversations, with tools provided via MCP (Model Context Protocol). The platform is [multi-tenant](tenancy.md) — all resources are scoped to a tenant. See [Authentication](authn.md) for identity types.

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
        Chat[Chat]
        Channels[Channels]
        Threads[Threads]
        Files[Files]
        TokenCounting[Token Counting]
        LLM[LLM]
        Secrets[Secrets]
        Notifications[Notifications]
        Runner[Runner]
        AgentState[Agent State]
        Tracing[Tracing]
        Authorization[Authorization]
        ZitiMgmt[Ziti Management]
    end

    subgraph Workloads
        Agent1[Agent Container]
        Agent2[Agent Container]
        MCP1[MCP Server]
        MCP2[MCP Server]
    end

    subgraph Monolith
        PS[platform-server]
    end

    WebApp & MobileApp -- "/apiv2/" --> Gateway
    WebApp & MobileApp -- "/api" --> PS
    ThirdParty <--> Channels

    Gateway --> ZitiMgmt
    Gateway --> Chat
    Gateway --> Files
    Gateway --> Notifications
    Gateway --> LLM
    Gateway --> Secrets
    Chat --> Threads
    Channels --> Threads

    Threads -->|publish events| Notifications
    AgentsOrch --> Runner
    AgentsOrch --> ZitiMgmt
    AgentsOrch --> Threads

    Runner --> Agent1 & Agent2
    Agent1 --> MCP1
    Agent2 --> MCP2
    Agent1 & Agent2 --> Threads
    Agent1 & Agent2 --> Notifications
    Agent1 & Agent2 --> AgentState
    Agent1 & Agent2 --> Files
    Agent1 & Agent2 --> TokenCounting
    Agent1 & Agent2 --> LLM
    Agent1 & Agent2 -.-> Tracing

    Chat --> Authorization
    Files --> Authorization
    Threads --> Authorization
    Teams --> Authorization
    Authorization --> OpenFGA[(OpenFGA)]

    Teams --> AgentsOrch
```

## Component Summary

| Component | Responsibility |
|-----------|---------------|
| **Chat** | Built-in web/mobile app chat experience. Thread lifecycle, unread counts. Built on top of Threads |
| **Channels** | Bidirectional interface connecting 3rd-party products (Slack, etc.) with Threads. Each channel creates and manages its own threads |
| **Threads** | Generic messaging between participants. Stores messages, tracks participants by ID, provides message acknowledgment. Participant-type-agnostic |
| **Files** | File upload, metadata storage, and pre-signed download URL generation. Backed by S3-compatible object storage |
| **Token Counting** | Per-message token counting for LLM messages |
| **LLM** | Manages LLM providers and models. Proxies LLM API calls from agents to providers with injected credentials |
| **Secrets** | Manages secret providers and secrets. Resolves secret values from external providers at runtime |
| **Notifications** | Real-time event fanout via persistent connections (socket). All services publish state change events through Notifications |
| **Authorization** | Fine-grained access control. Thin proxy to OpenFGA — centralizes configuration, adds observability. Services call Authorization for permission checks and relationship writes |
| **[Agents Orchestrator](agents-orchestrator.md)** | Reconciles agent workloads for threads with unacknowledged messages |
| **Agent State** | Long-term agent context persistence (APSS) |
| **Tracing** | Ingestion and query of tracing data. Extended OpenTelemetry protocol for real-time in-progress events |
| **Teams** | Management of team resources: agents, workspaces, MCP servers, etc. |
| **Runner** | Executes workloads. Implementations: docker-runner, k8s-runner |
| **Gateway** | Exposes platform methods for external usage. Accessible at `gateway.agyn.dev` (subdomain) and `agyn.dev/apiv2/` (path-based, prefix stripped) |
| **Ziti Management** | Manages OpenZiti identities, services, and policies. Encapsulates all OpenZiti Controller API interactions |

## Data Stores

| Store | Current Usage |
|-------|--------------|
| PostgreSQL | Primary relational store (agent state, platform data) |
| Redis | Pub/sub for notifications, caching |
| Filesystem | Graph store (agent graph definitions persisted as filesystem dataset) |
| Object Storage (S3) | Media file storage (MinIO locally, any S3-compatible in production) |
| OpenFGA | Relationship-based access control (authorization model and relationship tuples). PostgreSQL-backed |

## Repository Map

| Repository | Contents | Language | Status |
|------------|----------|----------|--------|
| `agynio/api` | API schemas: protobuf (internal gRPC) and OpenAPI (external) | Proto, YAML | Active |
| `agynio/platform` | Monolith: platform-server, docker-runner, LLM package, platform-ui | TypeScript | Active (being decomposed) |
| `agynio/notifications` | Notifications service | Go | Standalone service |
| `agynio/gateway` | Gateway service | Go | Standalone service |
| `agynio/agent-state` | Agent State (APSS) service | Go | Standalone service |
| `agynio/openfga-model` | OpenFGA authorization model and Terraform module | DSL, HCL | Planned |
| `agynio/authorization` | Authorization service (thin proxy to OpenFGA) | Go | Planned |
| `agynio/agyn-cli` | Platform CLI — Gateway API access | Go | Planned |
| `agynio/agynd-cli` | Agent wrapper daemon — bridges agent CLIs with platform | Go | Planned |
| `agynio/agn-cli` | Agent loop implementation — LLM reasoning with tool use | Go | Planned |
| `agynio/k8s-runner` | Kubernetes-native Runner implementation | Go | Planned |
| `agynio/architecture` | This documentation | Markdown | — |
