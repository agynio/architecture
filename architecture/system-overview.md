# System Overview

Agyn is a Kubernetes-native AI agent orchestrator. It manages the lifecycle of AI agents that communicate with humans and each other through threaded conversations, with tools provided via MCP (Model Context Protocol). The platform uses [organizations](organizations.md) to group configuration resources, with [ReBAC](authz.md) for fine-grained access control on all resources. See [Authentication](authn.md) for identity types.

## Component Diagram

```mermaid
graph TB
    subgraph External
        WebApp[Web App]
        MobileApp[Mobile App]
        ThirdParty[3rd-party Apps<br/>Slack, etc.]
    end

    subgraph Control Plane
        Agents[Agents]
        AgentsOrch[Agents<br/>Orchestrator]
        Organizations[Organizations]
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
        Users[Users]
        Identity[Identity]
        AppsService[Apps Service]
        LLMProxy[LLM Proxy]
    end

    subgraph Apps
        RemindersApp[Reminders]
    end

    subgraph Workloads
        Agent1[Agent Container]
        Agent2[Agent Container]
        MCP1[MCP Server]
        MCP2[MCP Server]
    end

    WebApp & MobileApp -- "/api/" --> Gateway
    ThirdParty <--> Channels

    Gateway --> ZitiMgmt
    Gateway --> Users
    Gateway --> Authorization
    Gateway --> Chat
    Gateway --> Files
    Gateway --> Notifications
    Gateway --> Secrets
    Gateway --> Threads
    Gateway --> AgentState
    Gateway --> TokenCounting
    Gateway --> Tracing
    LLMProxy --> LLM
    LLMProxy --> ZitiMgmt
    LLMProxy --> Users
    LLMProxy --> Authorization
    Chat --> Threads
    Chat --> Identity
    Chat --> Users
    Channels --> Threads

    Users --> Identity
    Agents --> Identity

    Threads -->|publish events| Notifications
    AgentsOrch --> Runner
    AgentsOrch --> ZitiMgmt
    AgentsOrch --> Threads

    Runner --> Agent1 & Agent2
    Agent1 --> MCP1
    Agent2 --> MCP2
    Agent1 & Agent2 -->|OpenZiti| Gateway
    Agent1 & Agent2 -->|OpenZiti| LLMProxy

    Chat --> Authorization
    Files --> Authorization
    Threads --> Authorization
    Agents --> Authorization
    Organizations --> Authorization
    Authorization --> OpenFGA[(OpenFGA)]

    AppsService --> Identity
    AppsService --> ZitiMgmt
    AppsService --> Authorization
    Gateway --> AppsService
    RemindersApp -->|OpenZiti| Gateway

    Agents --> AgentsOrch
```

## Component Summary

| Component | Responsibility |
|-----------|---------------|
| **Identity** | Central identity registry. Maps `identity_id` to `identity_type` for all identity types |
| **Users** | User identity records and profiles. Provisions users on first OIDC login, serves profiles for display |
| **Organizations** | Organization lifecycle (CRUD) and listing accessible organizations for an identity (queries Authorization for organization IDs, enriches with organization details) |
| **Chat** | Built-in web/mobile app chat experience. Thread lifecycle, unread counts. Built on top of Threads |
| **Channels** | Bidirectional interface connecting 3rd-party products (Slack, etc.) with Threads. Each channel creates and manages its own threads |
| **Threads** | Generic messaging between participants. Stores messages, tracks participants by ID, provides message acknowledgment. Participant-type-agnostic |
| **Files** | File upload, metadata storage, and pre-signed download URL generation. Backed by S3-compatible object storage |
| **Token Counting** | Per-message token counting for LLM messages |
| **LLM** | Manages LLM providers and models. Provides model resolution (model ID → provider endpoint, token, remote name) for the LLM Proxy |
| **[LLM Proxy](llm-proxy.md)** | Exposes an OpenAI-compatible Responses API endpoint for agents. Authenticates callers, resolves models via LLM service, forwards requests to external providers |
| **Secrets** | Manages secret providers and secrets. Resolves secret values from external providers at runtime |
| **Notifications** | Real-time event fanout via persistent connections (socket). All services publish state change events through Notifications |
| **Authorization** | Fine-grained access control. Thin proxy to OpenFGA — centralizes configuration, adds observability. Services call Authorization for permission checks and relationship writes |
| **[Agents Orchestrator](agents-orchestrator.md)** | Reconciles agent workloads for threads with unacknowledged messages |
| **Agent State** | Long-term agent context persistence (APSS) |
| **Tracing** | Span ingestion and query. Implements standard OTLP TraceService/Export with upsert semantics for in-progress spans |
| **[Agents](agents-service.md)** | Management of agent resources: agents, volumes, MCP servers, skills, hooks, etc. |
| **Runner** | Executes workloads. Current implementation: [k8s-runner](k8s-runner.md) |
| **Gateway** | Exposes platform methods for external usage via [ConnectRPC](gateway.md#connectrpc) (gRPC + HTTP/JSON). Accessible at `gateway.agyn.dev` (subdomain) and `agyn.dev/api/` (path-based, prefix stripped) |
| **Ziti Management** | Manages OpenZiti identities, services, and policies. Encapsulates all OpenZiti Controller API interactions |
| **[Apps Service](apps-service.md)** | App registration, profiles, and enrollment. Manages the lifecycle of [apps](apps.md) |
| **[Reminders](apps/reminders.md)** | Platform-provided [app](apps.md). Delivers delayed messages to threads on behalf of agents |

## Data Stores

| Store | Current Usage |
|-------|--------------|
| PostgreSQL | Primary relational store (agent state, platform data, user records, identity registry, organizations) |
| Redis | Pub/sub for notifications, caching |
| Filesystem | Graph store (agent graph definitions persisted as filesystem dataset) |
| Object Storage (S3) | Media file storage (MinIO locally, any S3-compatible in production) |
| OpenFGA | Relationship-based access control (authorization model and relationship tuples). PostgreSQL-backed |

## Repository Map

| Repository | Contents | Language | Status |
|------------|----------|----------|--------|
| `agynio/api` | API schemas: protobuf (internal gRPC + external gateway ConnectRPC) | Proto | Active |
| `agynio/notifications` | Notifications service | Go | Standalone service |
| `agynio/gateway` | Gateway service | Go | Standalone service |
| `agynio/agent-state` | Agent State (APSS) service | Go | Standalone service |
| `agynio/tracing` | Tracing service — span ingestion and query | Go | Planned |
| `agynio/authorization` | Authorization service (thin proxy to OpenFGA) + authorization model & Terraform | Go, DSL, HCL | Planned |
| `agynio/identity` | Identity registry service | Go | Planned |
| `agynio/users` | Users service | Go | Planned |
| `agynio/organizations` | Organizations service | Go | Planned |
| `agynio/agents` | Agents service (agent resource management) | Go | Planned |
| `agynio/agyn-cli` | Platform CLI — Gateway API access | Go | Planned |
| `agynio/agynd-cli` | Agent wrapper daemon — bridges agent CLIs with platform | Go | Planned |
| `agynio/codex-sdk-go` | Go client library for Codex CLI | Go | Active |
| `agynio/agn-cli` | Agent loop implementation — LLM reasoning with tool use | Go | Planned |
| `agynio/llm-proxy` | LLM Proxy — OpenAI-compatible Responses API endpoint for agents | Go | Planned |
| `agynio/agent-init-codex` | Init container image: agynd + Codex CLI | Dockerfile | Active |
| `agynio/agent-init-claude` | Init container image: agynd + Claude Code CLI | Dockerfile | Active |
| `agynio/agent-init-agn` | Init container image: agynd + agn CLI | Dockerfile | Active |
| `agynio/k8s-runner` | Kubernetes-native Runner implementation | Go | Planned |
| `agynio/terraform-provider-agyn` | Terraform provider for agent resource management | Go | Planned |
| `agynio/apps` | Apps Service | Go | Planned |
| `agynio/reminders` | Reminders app | Go | Planned |
| `agynio/architecture` | This documentation | Markdown | — |
