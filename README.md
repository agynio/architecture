# Agyn Architecture

Architecture documentation for the Agyn AI agent orchestrator platform.

## Documents

| Document | Description |
|----------|-------------|
| [System Overview](system-overview.md) | High-level view of the platform, core components, and their responsibilities |
| [Control Plane & Data Plane](control-data-plane.md) | Boundary definitions, criteria, and service mapping |
| [Agent Architecture](agent.md) | Agent structure, LLM loop, summarization, state, tools, and interface contract |
| [Channels](channels.md) | Channel interface, bidirectional message flow, and 3rd-party integration |
| [Threads](threads.md) | Messaging service interface and data model |
| [Notifications](notifications.md) | Real-time notification service, pub/sub model, and transport |
| [Runner](runner.md) | Workload execution service and container lifecycle |
| [Agent State](agent-state.md) | Long-term agent context persistence service |
| [Teams](teams.md) | Team resource management |
| [Tracing](tracing.md) | Tracing ingestion and query service |
| [Gateway](gateway.md) | External API gateway |
| [API Contracts](api-contracts.md) | Schema management, internal gRPC and external OpenAPI conventions |
| [Open Questions](open-questions.md) | Unresolved architectural decisions and topics for discussion |
