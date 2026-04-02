# Agyn

Product and architecture documentation for the Agyn AI agent orchestrator platform.

## How to Use This Documentation

### `/product` — Product Source of Truth

The desired state of the product. Declarative descriptions of capabilities, behaviors, and user experiences. No references to current implementation state, no transitional statements, no legacy caveats. When the product changes, this directory is updated to reflect the new target — not annotated with "currently X, will become Y."

### `/architecture` — Architecture Source of Truth

The desired state of the system. Declarative descriptions of all services, patterns, protocols, and contracts. No references to current implementation state, no transitional statements, no legacy caveats. When the architecture changes, this directory is updated to reflect the new target — not annotated with "currently X, will become Y."

### `/changes` — Pending Deltas

Gaps between the desired state described in `/product` or `/architecture` and current reality. Each file represents a single delta. When reality matches the desired state, the file is deleted. Git history preserves the full record.

### `/open-questions.md` — Unresolved Decisions

Topics that require further investigation and a decision. Specifications for areas with unresolved questions should be avoided. Raise an open question or wait for a decision instead of specifying behavior that may change.

## Structure

### [Product](product/) — Desired Product State

Target product definition. Describes how the product should work from the user's perspective.

| Document | Description |
|----------|-------------|
| [Overview](product/overview.md) | Product vision, value proposition, target users, principles |
| [Concepts](product/concepts.md) | Domain glossary — canonical definitions of product terms |
| **Chat** | |
| [Chat](product/chat/chat.md) | Conversations with users and agents |
| **Console** | |
| [Console](product/console/console.md) | Management UI — organizations, users, agents, providers, runners, monitoring |
| **Tracing** | |
| [Run Timeline](product/tracing/run-timeline.md) | Observability view for a single agent run |

### [Architecture](architecture/) — Desired Architecture State

Target architecture of the platform. Describes how the system should work.

| Document | Description |
|----------|-------------|
| [System Overview](architecture/system-overview.md) | Components, responsibilities, data stores, repository map |
| [Organizations](architecture/organizations.md) | Organization model, Organizations service, resource scoping (org-scoped vs independent), ReBAC access control |
| [Identity](architecture/identity.md) | Central identity type registry |
| [Users](architecture/users.md) | User identity records, profiles, OIDC provisioning |
| [Authentication](architecture/authn.md) | Identity types, OIDC, agent network auth, service tokens |
| [Authorization](architecture/authz.md) | Fine-grained access control via OpenFGA. Authorization service, model, deployment |
| [API Tokens](architecture/api-tokens.md) | Long-lived opaque tokens for programmatic access (CI, integrations, developer tooling) |
| [Control Plane & Data Plane](architecture/control-data-plane.md) | Boundary definitions, criteria, service classification |
| [Resource Definitions](architecture/resource-definitions.md) | Canonical schemas for all agent-managed resources |
| [Agent](architecture/agent/) | Agent contract, our implementation, disk-based state persistence |
| [agyn-cli](architecture/agyn-cli.md) | Platform CLI — Gateway API access for admins, developers, and agents |
| [agynd-cli](architecture/agynd-cli.md) | Agent wrapper daemon — bridges agent CLIs with platform services |
| [Agent Init Container](architecture/agent-init.md) | Init container design for injecting platform binaries into agent pods |
| [agn-cli](architecture/agn-cli.md) | Our agent loop implementation — LLM reasoning with tool use |
| [Chat](architecture/chat.md) | Built-in web/mobile app chat on top of Threads |
| [Console](architecture/console.md) | Management UI — separate SPA at `console.agyn.dev`, role-based visibility, Gateway API consumer |
| [Apps](architecture/apps.md) | Apps concept — services that interact with threads (Reminders, Slack, GitHub) |
| [Apps Service](architecture/apps-service.md) | App registration, profiles, and enrollment |
| [Reminders](architecture/apps/reminders.md) | Platform-provided app — delayed messages to threads |
| [Threads](architecture/threads.md) | Messaging service interface and data model |
| [Media](architecture/media.md) | File attachments in thread messages |
| [Token Counting](architecture/token-counting.md) | Per-message token counting service |
| [Providers, Models, and Secrets](architecture/providers.md) | LLM provider / model and secret provider / secret resource ownership |
| [LLM](architecture/llm.md) | LLM provider/model management and LLM call proxy |
| [Secrets](architecture/secrets.md) | Secret provider/secret management and secret resolution |
| [Notifications](architecture/notifications.md) | Real-time event fanout service |
| [Runners](architecture/runners.md) | Runner registration and workload runtime state |
| [Runner](architecture/runner.md) | Workload execution service |
| [k8s-runner](architecture/k8s-runner.md) | Kubernetes-native Runner implementation |
| [Agents Orchestrator](architecture/agents-orchestrator.md) | Agent workload reconciliation — start, monitor, stop |
| [Agents Service](architecture/agents-service.md) | Agent resource management |
| [Tracing](architecture/tracing.md) | Span ingestion and query — standard OTLP with upsert for in-progress spans. Captures full LLM call context |
| [Gateway](architecture/gateway.md) | External API surface — ConnectRPC (gRPC + HTTP/JSON) |
| [API Contracts](architecture/api-contracts.md) | Proto schema conventions for internal and external APIs |

### [Operations](architecture/operations/) — CI/CD, Local Development, Configuration

How services are built, deployed, run locally, and configured.

| Document | Description |
|----------|-------------|
| [CI/CD](architecture/operations/ci-cd.md) | Image and Helm chart publishing via GitHub Actions |
| [Local Development](architecture/operations/local-development.md) | Bootstrap cluster and DevSpace inner loop |
| [Terraform Provider](architecture/operations/terraform-provider.md) | Recommended configuration-as-code interface |
| [New Service Development](architecture/operations/new-service.md) | End-to-end process: API schema → implementation → CI/CD → bootstrap → E2E tests |
| [E2E Testing](architecture/operations/e2e-testing.md) | In-cluster E2E tests, DevSpace test pods, deterministic LLM via TestLLM |

### [Changes](changes/) — Pending Deltas

Gaps between desired state and current reality. See [Changes README](changes/README.md) for lifecycle rules.

### [Open Questions](open-questions.md)

Unresolved product and architectural decisions requiring discussion.
