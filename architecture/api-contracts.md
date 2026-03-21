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
| Agents | `agynio/api/agents/v1/agents.proto` |
| Runner | `agynio/api/runner/v1/runner.proto` |
| Notifications | `agynio/api/notifications/v1/notifications.proto` |
| Files | `agynio/api/files/v1/files.proto` |
| Agent State | `agynio/api/agent_state/v1/agent_state.proto` |
| Threads | `agynio/api/threads/v1/threads.proto` |
| Chat | `agynio/api/chat/v1/chat.proto` |
| Tracing | `agynio/api/tracing/v1/tracing.proto` |

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
| ThreadsGateway | `agynio/api/gateway/v1/threads.proto` | [Threads](threads.md) |
| ChatGateway | `agynio/api/gateway/v1/chat.proto` | [Chat](chat.md) |
| NotificationsGateway | `agynio/api/gateway/v1/notifications.proto` | [Notifications](notifications.md) |
| FilesGateway | `agynio/api/gateway/v1/files.proto` | [Files](media.md) |
| AgentStateGateway | `agynio/api/gateway/v1/agent_state.proto` | [Agent State](agent/state.md) |
| TokenCountingGateway | `agynio/api/gateway/v1/token_counting.proto` | [Token Counting](token-counting.md) |
| LLMGateway | `agynio/api/gateway/v1/llm.proto` | [LLM](llm.md) |
| TracingGateway | `agynio/api/gateway/v1/tracing.proto` | [Tracing](tracing.md) |
| SecretsGateway | `agynio/api/gateway/v1/secrets.proto` | [Secrets](secrets.md) |

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
