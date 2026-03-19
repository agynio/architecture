# Migration Roadmap

Steps to move from the current monolith (`agynio/platform`) to the target architecture.

## Migration Phases

```mermaid
graph LR
    subgraph "Phase 1: Already Done"
        S1[Gateway]
        S2[Agent State]
        S3[Notifications]
        S4[Docker Runner]
    end

    subgraph "Phase 2: Extract from Monolith"
        S5[Agent → standalone container]
        S6[Threads → standalone service]
        S7[Teams → standalone service]
    end

    subgraph "Phase 3: New Services"
        S8[Channels service]
        S9[Agents orchestrator]
        S10[Tracing service]
        S11[k8s-runner]
        S12[Files service]
    end

    S1 --> S5
    S2 --> S5
    S3 --> S5
    S4 --> S5

    S5 --> S6 --> S7
    S7 --> S8 --> S9
    S7 --> S12
    S9 --> S10 & S11
```

## Phase 1 — Standalone Services (Done)

These services are already extracted and running independently:

| Service | Repo | Notes |
|---------|------|-------|
| Gateway | `agynio/gateway` | Serves Team API, proxies remaining to monolith |
| Agent State (APSS) | `agynio/agent-state` | gRPC service with PostgreSQL |
| Notifications | `agynio/notifications` | gRPC + Socket.IO, Redis pub/sub |
| Docker Runner | `agynio/platform` (separate deploy) | gRPC, HMAC auth, Helm chart |

## Phase 2 — Extract from Monolith

### Agent Extraction

Extract the agent from `platform-server` into a standalone container:

- **Move**: `packages/llm/` (Loop, Reducer, Router, FunctionTool, messages) and agent node logic from `packages/platform-server/src/nodes/agent/`.
- **Connect**: Agent State service (remote gRPC) for state persistence.
- **Connect**: MCP servers for tools.
- **Implement**: Pull + notifications communication protocol and CLI wrapper. See [agent overview](../architecture/agent/overview.md#communication-protocol).

### Threads Extraction

Extract messaging from `platform-server` into a standalone Threads service:

- **Define**: gRPC API for Threads (CreateThread, ArchiveThread, AddParticipant, SendMessage, GetThreads, GetMessages).
- **Move**: Thread/message persistence from platform-server to dedicated service.
- **Update**: Gateway to route thread operations to the new service.

### Teams Extraction

Complete the extraction of team resource management:

- **Move**: Resource CRUD from platform-server to a standalone Teams service with its own data store.
- **Update**: Gateway to route directly to Teams service (remove platform-server proxy).

## Phase 3 — New Services

### Channels Service

New service implementing the channel interface:

- Bidirectional message translation (external ↔ threads).
- Channel configuration management (control plane side).
- Live connections to 3rd-party APIs (data plane side).
- Replace Slack trigger node from platform-server.

### Agents Orchestrator

New control plane service:

- Watch for threads with pending messages.
- Reconcile agent workloads via Runner.
- Manage agent lifecycle (start, monitor, stop).

### Tracing Service

New service replacing the removed tracing stack:

- Extended OpenTelemetry protocol for real-time in-progress events.
- Ingestion and query APIs.

### Files Service

New data plane service for file attachments:

- gRPC API for upload, metadata, and download URLs.
- Object storage integration with metadata persistence.
- Gateway streams multipart uploads to the gRPC client-streaming API.

### k8s-runner

Kubernetes-native Runner implementation. See [k8s-runner architecture](../architecture/k8s-runner.md).

- Same gRPC interface as docker-runner.
- Creates Pods instead of Docker containers.
- Persistent volumes backed by PVCs, ephemeral volumes by emptyDir.

## Gateway Evolution

As services are extracted, the Gateway evolves:

| Phase | Gateway Behavior |
|-------|-----------------|
| Current | Team API validated locally; everything else proxied to monolith |
| Phase 2 | Routes threads, teams to their standalone services |
| Phase 3 | Routes all traffic to individual services; monolith proxy removed |
