# API Contracts

## Schema Repository

All API schemas are stored in `agynio/api`.

Services **do not** commit generated schema code in their own repositories. They consume schemas as dependencies at build time.

## Internal API — gRPC

| Aspect | Details |
|--------|---------|
| Protocol | gRPC (HTTP/2) |
| IDL | Protocol Buffers (proto3) |
| Tooling | [Buf](https://buf.build) for linting, breaking change detection, and publishing |
| Registry | `buf.build/agynio/api` |
| Module path | `proto/` in `agynio/api` |

### Proto Services

| Service | Proto Path |
|---------|-----------|
| [Agents](agents-service.md) | `agynio/api/agents/v1/agents.proto` |
| [Agent State](agent/state.md) | `agynio/api/agent_state/v1/agent_state.proto` |
| [Apps](apps-service.md) | `agynio/api/apps/v1/apps.proto` |
| [Authorization](authz.md) | `agynio/api/authorization/v1/authorization.proto` |
| [Chat](chat.md) | `agynio/api/chat/v1/chat.proto` |
| [Egress Rules](egress-rules-service.md) | `agynio/api/egress/v1/egress.proto` |
| [Expose](expose-service.md) | `agynio/api/expose/v1/expose.proto` |
| [Files](media.md) | `agynio/api/files/v1/files.proto` |
| [Groups](groups-service.md) | `agynio/api/groups/v1/groups.proto` |
| [Identity](identity.md) | `agynio/api/identity/v1/identity.proto` |
| [Image Proxy](image-proxy.md) | `agynio/api/image_proxy/v1/image_proxy.proto` |
| [Images](images-service.md) | `agynio/api/images/v1/images.proto` |
| [LLM](llm.md) | `agynio/api/llm/v1/llm.proto` |
| [Metering](metering.md) | `agynio/api/metering/v1/metering.proto` |
| [Networks](networks-service.md) | `agynio/api/networks/v1/networks.proto` |
| [Notifications](notifications.md) | `agynio/api/notifications/v1/notifications.proto` |
| [Organizations](organizations.md) | `agynio/api/organizations/v1/organizations.proto` |
| [Runner](runner.md) | `agynio/api/runner/v1/runner.proto` |
| [Runners](runners.md) | `agynio/api/runners/v1/runners.proto` |
| [Secrets](secrets.md) | `agynio/api/secrets/v1/secrets.proto` |
| [Terminal Proxy](terminal-proxy.md) | `agynio/api/terminal_proxy/v1/terminal_proxy.proto` |
| [Threads](threads.md) | `agynio/api/threads/v1/threads.proto` |
| [Token Counting](token-counting.md) | `agynio/api/token_counting/v1/token_counting.proto` |
| [Tracing](tracing.md) | `agynio/api/tracing/v1/tracing.proto` |
| [Users](users.md) | `agynio/api/users/v1/users.proto` |
| [Ziti Management](openziti.md) | `agynio/api/ziti_management/v1/ziti_management.proto` |

### Conventions

- Package naming: `agynio.api.<service>.v1`
- Go package: `github.com/agynio/api/gen/agynio/api/<service>/v1;<service>v1`
- Buf lint: `STANDARD`
- Breaking change detection: `FILE`

## External API — ConnectRPC

The external API is defined by **gateway proto services** in `agynio/api`. These proto services describe only the methods exposed through the [Gateway](gateway.md). They import and reuse message types from internal service protos.

| Aspect | Details |
|--------|---------|
| Protocol | [ConnectRPC](https://connectrpc.com/) — serves Connect (HTTP/JSON), gRPC, and gRPC-Web from the same handler |
| IDL | Protocol Buffers (proto3) — same as internal API |
| Tooling | Buf + `protoc-gen-connect-go` |
| Registry | `buf.build/agynio/api` (same module as internal protos) |

### Gateway Proto Services

| Gateway Service | Proto Path | Internal Service |
|----------------|-----------|-----------------|
| AgentsGateway | `agynio/api/gateway/v1/agents.proto` | [Agents](agents-service.md) |
| AgentStateGateway | `agynio/api/gateway/v1/agent_state.proto` | [Agent State](agent/state.md) |
| AppsGateway | `agynio/api/gateway/v1/apps.proto` | [Apps](apps-service.md) |
| ChatGateway | `agynio/api/gateway/v1/chat.proto` | [Chat](chat.md) |
| EgressRulesGateway | `agynio/api/gateway/v1/egress.proto` | [EgressRules](egress-rules-service.md) |
| ExposeGateway | `agynio/api/gateway/v1/expose.proto` | [Expose](expose-service.md) |
| FilesGateway | `agynio/api/gateway/v1/files.proto` | [Files](media.md) |
| GroupsGateway | `agynio/api/gateway/v1/groups.proto` | [Groups](groups-service.md) |
| ImagesGateway | `agynio/api/gateway/v1/images.proto` | [Images](images-service.md) |
| LLMGateway | `agynio/api/gateway/v1/llm.proto` | [LLM](llm.md) |
| MeteringGateway | `agynio/api/gateway/v1/metering.proto` | [Metering](metering.md) |
| NetworksGateway | `agynio/api/gateway/v1/networks.proto` | [Networks](networks-service.md) |
| NotificationsGateway | `agynio/api/gateway/v1/notifications.proto` | [Notifications](notifications.md) |
| OrganizationsGateway | `agynio/api/gateway/v1/organizations.proto` | [Organizations](organizations.md) |
| RunnersGateway | `agynio/api/gateway/v1/runners.proto` | [Runners](runners.md) |
| SecretsGateway | `agynio/api/gateway/v1/secrets.proto` | [Secrets](secrets.md) |
| TerminalGateway | `agynio/api/gateway/v1/terminal.proto` | [Terminal Proxy](terminal-proxy.md) |
| ThreadsGateway | `agynio/api/gateway/v1/threads.proto` | [Threads](threads.md) |
| TokenCountingGateway | `agynio/api/gateway/v1/token_counting.proto` | [Token Counting](token-counting.md) |
| TracingGateway | `agynio/api/gateway/v1/tracing.proto` | [Tracing](tracing.md) |
| UsersGateway | `agynio/api/gateway/v1/users.proto` | [Users](users.md) |

### How It Works

Gateway proto services define a curated subset of methods for external use. Message types are imported from internal protos — not duplicated:

```proto
// agynio/api/gateway/v1/agents.proto
syntax = "proto3";
package agynio.api.gateway.v1;

import "agynio/api/agents/v1/agents.proto";

service AgentsGateway {
  rpc CreateAgent(agynio.api.agents.v1.CreateAgentRequest)
      returns (agynio.api.agents.v1.CreateAgentResponse);
  // ... other externally-exposed methods
}
```

Internal-only methods are excluded by not listing them in the gateway proto service.

### Conventions

- Gateway proto package: `agynio.api.gateway.v1`
- Gateway proto path: `proto/agynio/api/gateway/v1/`
- Message types: imported from internal protos, not redefined
- Buf lint and breaking change detection: same rules as internal protos
