# Agyn Architecture

Architecture documentation for the Agyn AI agent orchestrator platform.

## Structure

### [Architecture](architecture/) — Desired State

Target architecture of the platform. Describes how the system should work.

| Document | Description |
|----------|-------------|
| [System Overview](architecture/system-overview.md) | Components, responsibilities, data stores, repository map |
| [Control Plane & Data Plane](architecture/control-data-plane.md) | Boundary definitions, criteria, service classification |
| [Agent](architecture/agent.md) | Agent structure, LLM loop, summarization, state, tools, interface |
| [Channels](architecture/channels.md) | Bidirectional interface for 3rd-party and own apps |
| [Threads](architecture/threads.md) | Messaging service interface and data model |
| [Notifications](architecture/notifications.md) | Real-time event fanout service |
| [Runner](architecture/runner.md) | Workload execution service |
| [Agent State](architecture/agent-state.md) | Long-term agent context persistence |
| [Teams](architecture/teams.md) | Team resource management |
| [Tracing](architecture/tracing.md) | Tracing ingestion and query service |
| [Gateway](architecture/gateway.md) | External API surface |
| [API Contracts](architecture/api-contracts.md) | gRPC and OpenAPI schema conventions |

### [Gaps](gaps/) — Current State vs. Desired & Roadmap

What is already implemented, what is missing, and the migration path.

| Document | Description |
|----------|-------------|
| [Current State](gaps/current-state.md) | Inventory of existing services and their status |
| [Migration Roadmap](gaps/migration-roadmap.md) | Steps to move from monolith to target architecture |

### [Open Questions](open-questions.md)

Unresolved architectural decisions requiring discussion.
