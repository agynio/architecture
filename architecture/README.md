# Architecture

Desired architecture state of the platform. These docs describe services, patterns, protocols, and contracts.

## Core documents

| Document | Description |
|----------|-------------|
| [System Overview](system-overview.md) | Components, responsibilities, data stores, repository map |
| [Organizations](organizations.md) | Organization model, Organizations service, members management (invites, direct membership, roles), resource scoping (org-scoped vs independent), ReBAC access control |
| [Identity](identity.md) | Central identity type registry |
| [Users](users.md) | User identity records, profiles, OIDC provisioning |
| [Authentication](authn.md) | Identity types, OIDC, agent network auth, service tokens |
| [Authorization](authz.md) | Fine-grained access control via OpenFGA. Authorization service, model, deployment |
| [API Tokens](api-tokens.md) | Long-lived opaque tokens for programmatic access (CI, integrations, developer tooling) |
| [Control Plane and Data Plane](control-data-plane.md) | Boundary definitions, criteria, service classification |
| [Resource Definitions](resource-definitions.md) | Canonical schemas for all agent-managed resources |
| [Agent](agent/) | Agent contract, our implementation, disk-based state persistence |
| [agyn-cli](agyn-cli.md) | Platform CLI - Gateway API access for admins, developers, and agents |
| [agynd-cli](agynd-cli.md) | Agent wrapper daemon - bridges agent CLIs with platform services |
| [Agent Init Container](agent-init.md) | Init container design for injecting platform binaries into agent pods |
| [agn-cli](agn-cli.md) | Our agent loop implementation - LLM reasoning with tool use |
| [Chat](chat.md) | Built-in web/mobile app chat on top of Threads |
| [Console](console.md) | Management UI - separate SPA at `console.agyn.dev`, role-based visibility, Gateway API consumer |
| [Apps](apps.md) | Apps concept - services that interact with threads (Reminders, Slack, GitHub) |
| [Apps Service](apps-service.md) | App registration, profiles, and enrollment |
| [Reminders](apps/reminders.md) | Platform-provided app - delayed messages to threads |
| [Threads](threads.md) | Messaging service interface and data model |
| [Media](media.md) | File attachments in thread messages |
| [Media Proxy](media-proxy.md) | Authenticated media proxy - external URLs and platform files for inline display |
| [files-mcp](files-mcp.md) | MCP server for agent file access |
| [Token Counting](token-counting.md) | Per-message token counting service |
| [Providers, Models, and Secrets](providers.md) | LLM provider / model and secret provider / secret resource ownership |
| [LLM](llm.md) | LLM provider/model management and LLM call proxy |
| [Secrets](secrets.md) | Secret provider/secret management and secret resolution |
| [Notifications](notifications.md) | Real-time event fanout service |
| [Runners](runners.md) | Runner registration and workload runtime state |
| [Runner](runner.md) | Workload execution service |
| [k8s-runner](k8s-runner.md) | Kubernetes-native Runner implementation |
| [Agents Orchestrator](agents-orchestrator.md) | Agent workload reconciliation - start, monitor, stop |
| [Agents Service](agents-service.md) | Agent resource management |
| [Tracing](tracing.md) | Span ingestion and query - standard OTLP with upsert for in-progress spans. Captures full LLM call context |
| [Gateway](gateway.md) | External API surface - ConnectRPC (gRPC + HTTP/JSON) |
| [API Contracts](api-contracts.md) | Proto schema conventions for internal and external APIs |

## Operations

| Document | Description |
|----------|-------------|
| [CI/CD](operations/ci-cd.md) | Image and Helm chart publishing via GitHub Actions |
| [Local Development](operations/local-development.md) | Bootstrap cluster and DevSpace inner loop |
| [Terraform Provider](operations/terraform-provider.md) | Recommended configuration-as-code interface |
| [New Service Development](operations/new-service.md) | End-to-end process: API schema -> implementation -> CI/CD -> bootstrap -> E2E tests |
| [E2E Testing](operations/e2e-testing.md) | In-cluster E2E tests, DevSpace test pods, deterministic LLM via TestLLM |

## Related product docs

- [Product documentation](../product/README.md)
- [Product to architecture map](../maps/product-to-architecture.md)
