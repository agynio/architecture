# Architecture

Desired architecture state of the platform.

## Reading paths

- Orientation: [System Overview](system-overview.md) -> [Control Plane and Data Plane](control-data-plane.md) -> [Gateway](gateway.md) -> [API Contracts](api-contracts.md)
- Surface deep dives: [Product to architecture map](../maps/product-to-architecture.md)
- Operations:
  - [CI/CD](operations/ci-cd.md)
  - [Local Development](operations/local-development.md)
  - [Terraform Provider](operations/terraform-provider.md)
  - [New Service Development](operations/new-service.md)
  - [E2E Testing](operations/e2e-testing.md)
  - [Database Migrations](operations/database-migrations.md)

## By domain

### Platform entrypoints

- [System Overview](system-overview.md)
- [Control Plane and Data Plane](control-data-plane.md)
- [Gateway](gateway.md)
- [API Contracts](api-contracts.md)

### Identity and access

- [Identity](identity.md)
- [Users](users.md)
- [Organizations](organizations.md)
- [Authentication](authn.md)
- [Authorization](authz.md)
- [API Tokens](api-tokens.md)

### Messaging and UI surfaces

- [Chat](chat.md)
- [Threads](threads.md)
- [Console](console.md)
- [Media](media.md)
- [Media Proxy](media-proxy.md)
- [Notifications](notifications.md)

### Agents and execution

- [Agents Service](agents-service.md)
- [Agents Orchestrator](agents-orchestrator.md)
- [Agent](agent/)
- [Resource Definitions](resource-definitions.md)
- [Runners](runners.md)
- [Runner](runner.md)
- [k8s-runner](k8s-runner.md)
- [Agent Init Container](agent-init.md)
- [agyn-cli](agyn-cli.md)
- [agynd-cli](agynd-cli.md)
- [agn-cli](agn-cli.md)

### LLM and secrets

- [LLM](llm.md)
- [LLM Proxy](llm-proxy.md)
- [Providers, Models, and Secrets](providers.md)
- [Secrets](secrets.md)

### Apps

- [Apps](apps.md)
- [Apps Service](apps-service.md)
- [Reminders](apps/reminders.md)

### MCP and networking

- [MCP](mcp.md)
- [files-mcp](files-mcp.md)
- [OpenZiti](openziti.md)
- [Expose Service](expose-service.md)

### Observability

- [Tracing](tracing.md)
- [Token Counting](token-counting.md)
- [Metering](metering.md)

### Operations

- [CI/CD](operations/ci-cd.md)
- [Local Development](operations/local-development.md)
- [Terraform Provider](operations/terraform-provider.md)
- [New Service Development](operations/new-service.md)
- [E2E Testing](operations/e2e-testing.md)
- [Database Migrations](operations/database-migrations.md)
