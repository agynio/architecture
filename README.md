# Agyn Architecture

Architecture documentation for the Agyn AI agent orchestrator platform.

## How to Use This Documentation

### `/architecture` — Source of Truth

The desired state of the system. Declarative descriptions of all services, patterns, protocols, and contracts. No references to current implementation state, no transitional statements, no legacy caveats. When the architecture changes, this directory is updated to reflect the new target — not annotated with "currently X, will become Y."

### `/gaps` — Change Log

A reminder of what needs to change in service implementations after an architecture update. Not a source of truth and not a task list. Each gap is cleared once the service implementation is aligned with the architecture. When sequential architecture changes happen, services may skip intermediate states — always implement the final state described in `/architecture`, not intermediate steps.

### `/open-questions` — Unresolved Decisions

Topics that require further investigation and a decision from a supervisor. Implementation of system parts that are not clearly described in the architecture should be avoided. Raise an open question or wait for a decision instead of building on assumptions that may change.

## Structure

### [Architecture](architecture/) — Desired State

Target architecture of the platform. Describes how the system should work.

| Document | Description |
|----------|-------------|
| [System Overview](architecture/system-overview.md) | Components, responsibilities, data stores, repository map |
| [Multi-Tenancy](architecture/tenancy.md) | Tenant model, Tenant service, resource scoping, data isolation |
| [Users](architecture/users.md) | User identity records, profiles, OIDC provisioning |
| [Authentication](architecture/authn.md) | Identity types, OIDC, agent network auth, service tokens |
| [Authorization](architecture/authz.md) | Fine-grained access control via OpenFGA. Authorization service, model, deployment |
| [Control Plane & Data Plane](architecture/control-data-plane.md) | Boundary definitions, criteria, service classification |
| [Resource Definitions](architecture/resource-definitions.md) | Canonical schemas for all team-managed resources |
| [Agent](architecture/agent/) | Agent contract, our implementation, state persistence |
| [agyn-cli](architecture/agyn-cli.md) | Platform CLI — Gateway API access for admins, developers, and agents |
| [agynd-cli](architecture/agynd-cli.md) | Agent wrapper daemon — bridges agent CLIs with platform services |
| [agn-cli](architecture/agn-cli.md) | Our agent loop implementation — LLM reasoning with tool use |
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
| [k8s-runner](architecture/k8s-runner.md) | Kubernetes-native Runner implementation |
| [Agents Orchestrator](architecture/agents-orchestrator.md) | Agent workload reconciliation — start, monitor, stop |
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

### [Gaps](gaps/) — Implementation Alignment

What needs to change in services to match the current architecture.

| Document | Description |
|----------|-------------|
| [k8s-runner](gaps/k8s-runner.md) | New service — full implementation needed |
| [OpenZiti SDK Embedding](gaps/openziti-sdk-embedding.md) | Orchestrator, Runner, Gateway: embed SDK, remove sidecar/tunneler |
| [Runner HMAC Removal](gaps/runner-hmac-removal.md) | Remove HMAC auth from docker-runner and Orchestrator |

### [Open Questions](open-questions.md)

Unresolved architectural decisions requiring discussion.
