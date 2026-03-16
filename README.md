# Agyn Architecture

Architecture documentation for the Agyn AI agent orchestrator platform.

## Structure

### [Architecture](architecture/) — Desired State

Target architecture of the platform. Describes how the system should work.

| Document | Description |
|----------|-------------|
| [System Overview](architecture/system-overview.md) | Components, responsibilities, data stores, repository map |
| [Multi-Tenancy](architecture/tenancy.md) | Tenant model, resource scoping, data isolation |
| [Authentication](architecture/authn.md) | Identity types, OIDC, agent network auth, service tokens |
| [Authorization](architecture/authz.md) | Fine-grained access control via OpenFGA. Authorization service, model, deployment |
| [Control Plane & Data Plane](architecture/control-data-plane.md) | Boundary definitions, criteria, service classification |
| [Resource Definitions](architecture/resource-definitions.md) | Canonical schemas for all team-managed resources |
| [Agent](architecture/agent/) | Agent contract, our implementation, state persistence |
| [Orchestrator](architecture/orchestrator.md) | Workload lifecycle — reconciles agents, MCP servers, and discovery signals |
| [MCP Adapter](architecture/mcp-adapter.md) | Standalone binary wrapping any MCP server — bridges stdio/HTTP to gRPC |
| [Chat](architecture/chat.md) | Built-in web/mobile app chat on top of Threads |
| [Channels](architecture/channels.md) | Bidirectional interface for 3rd-party and own apps |
| [Threads](architecture/threads.md) | Messaging service interface and data model |
| [Media](architecture/media.md) | File attachments in thread messages |
| [Token Counting](architecture/token-counting.md) | Per-message token counting service |
| [Providers, Models, and Secrets](architecture/providers.md) | LLM provider / model and secret provider / secret resource ownership |
| [LLM](architecture/llm.md) | LLM provider/model management and LLM call proxy |
| [Secrets](architecture/secrets.md) | Secret provider/secret management and secret resolution |
| [Notifications](architecture/notifications.md) | Real-time event fanout service |
| [Runner](architecture/runner.md) | Workload execution service |
| [Teams](architecture/teams.md) | Team resource management |
| [Tracing](architecture/tracing.md) | Tracing ingestion and query service |
| [Gateway](architecture/gateway.md) | External API surface |
| [API Contracts](architecture/api-contracts.md) | gRPC and OpenAPI schema conventions |

### [Operations](architecture/operations/) — CI/CD, Local Development, Configuration

How services are built, deployed, run locally, and configured.

| Document | Description |
|----------|-------------|
| [CI/CD](architecture/operations/ci-cd.md) | Image and Helm chart publishing via GitHub Actions |
| [Local Development](architecture/operations/local-development.md) | Bootstrap cluster and DevSpace inner loop |
| [Terraform Provider](architecture/operations/terraform-provider.md) | Recommended configuration-as-code interface |
| [New Service Development](architecture/operations/new-service.md) | End-to-end process: API schema → implementation → CI/CD → bootstrap → E2E tests |

### [Gaps](gaps/) — Current State vs. Desired & Roadmap

What is already implemented, what is missing, and the migration path.

| Document | Description |
|----------|-------------|
| [Current State](gaps/current-state.md) | Inventory of existing services and their status |
| [Migration Roadmap](gaps/migration-roadmap.md) | Steps to move from monolith to target architecture |

### [Open Questions](open-questions.md)

Unresolved architectural decisions requiring discussion.
