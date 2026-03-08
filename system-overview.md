# System Overview

Agyn is a Kubernetes-native AI agent orchestrator. It manages the lifecycle of AI agents that communicate with humans and each other through threaded conversations, with tools provided via MCP (Model Context Protocol).

## Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                       External Clients                      │
│              (Web App, Mobile App, 3rd-party)               │
└──────────────┬──────────────────────────────┬───────────────┘
               │                              │
               ▼                              ▼
        ┌─────────────┐              ┌──────────────┐
        │   Gateway    │              │   Channels   │
        │ (External API│              │ (Slack, etc.)│
        └──────┬──────┘              └──────┬───────┘
               │                            │
               ▼                            ▼
        ┌─────────────────────────────────────────────┐
        │                  Threads                     │
        │         (Conversation messaging)             │
        └──────┬──────────────────────────┬───────────┘
               │                          │
               ▼                          ▼
        ┌─────────────┐          ┌────────────────┐
        │ Notifications│          │     Agents     │
        │ (Real-time   │          │ (Orchestrator) │
        │  fanout)     │          └───────┬────────┘
        └─────────────┘                   │
                                          ▼
                              ┌───────────────────────┐
                              │     Runner(s)          │
                              │ (docker-runner,        │
                              │  k8s-runner, etc.)     │
                              └───────────┬────────────┘
                                          │
                                ┌─────────┼──────────┐
                                ▼         ▼          ▼
                          ┌────────┐ ┌────────┐ ┌────────┐
                          │ Agent  │ │ Agent  │ │ Agent  │
                          │Container│ │Container│ │Container│
                          └───┬────┘ └───┬────┘ └───┬────┘
                              │          │          │
                              ▼          ▼          ▼
                        ┌──────────┐ ┌──────────┐
                        │Agent-State│ │ Tracing  │
                        └──────────┘ └──────────┘
```

## Component Summary

| Component | Responsibility |
|-----------|---------------|
| **Channels** | Bidirectional interface connecting 3rd-party products (Slack, etc.) and own apps (web, mobile) with Threads |
| **Threads** | Conversation messaging service. Multi-participant (humans and agents) |
| **Notifications** | Real-time event fanout. Holds persistent connections (socket) and delivers events to relevant clients |
| **Agents** | Orchestrator that spins up new agent workloads for threads with pending messages |
| **Agent-State** | Long-term agent context persistence |
| **Tracing** | Ingestion and query of tracing data. Extended OpenTelemetry protocol supporting real-time in-progress events |
| **Teams** | Management of team resources: agents, workspaces, MCP servers, etc. |
| **Runner** | Executes workloads. Implementations: docker-runner, k8s-runner |
| **Gateway** | Exposes methods for external usage |

## Data Stores

| Store | Current Usage |
|-------|--------------|
| PostgreSQL | Primary relational store |
| Redis | Pub/sub for notifications, caching |
| Filesystem | Graph store (agent graph definitions persisted as filesystem dataset) |

## Repository Map

| Repository | Contents |
|------------|----------|
| `agynio/api` | API schemas: protobuf (internal gRPC) and OpenAPI (external) |
| `agynio/platform` | Current monolith: platform-server, docker-runner, LLM package, platform-ui |
| `agynio/notifications` | Notifications service (Go) |
| `agynio/architecture` | This documentation |
