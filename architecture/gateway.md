# Gateway

## Overview

The Gateway exposes platform methods for external usage. It is the entry point for external clients (web app, mobile app, integrators, agents, channels) to interact with platform services.

The gateway is accessible via two ingress paths:

| Path | Host | Prefix stripped | Use case |
|------|------|-----------------|----------|
| Subdomain | `gateway.agyn.dev` | No | Direct access, service-to-service |
| Path-based | `agyn.dev/api/` | Yes (`/api/` → `/`) | UI consumption (same origin as the web app) |

The path-based route allows the web app (platform-ui) to call gateway APIs without cross-origin requests.

## Responsibilities

- Route external API requests to internal services.
- Serve both gRPC and HTTP/JSON protocols from the same handler using [ConnectRPC](#connectrpc).
- Stream multipart file uploads to FilesService.UploadFile (client-streaming gRPC).
- Authenticate requests and resolve identity context. For OIDC users: validate the `access_token` JWT signature against the IdP's JWKS endpoint, extract the `sub` claim, and resolve identity via [Users](users.md) service (`ResolveUser` / `CreateUser`). For OpenZiti actors: resolve identity via [Ziti Management](openziti.md). The Gateway has no session state — every request is authenticated independently via the bearer token. The Gateway does not validate organization membership — that responsibility belongs to the [authorization model](authz.md), checked by the service performing the operation. See [Authentication](authn.md) and [Organizations — Request Flow](organizations.md#request-flow).

## ConnectRPC

The Gateway uses [ConnectRPC](https://connectrpc.com/) (`connectrpc/connect-go`) to serve the external API. ConnectRPC is an open-source library (Apache 2.0) from the Buf team that serves three wire protocols from a single handler:

| Protocol | Transport | Content-Type | Primary audience |
|----------|-----------|-------------|-----------------|
| **Connect** | HTTP/1.1 or HTTP/2, plain JSON bodies | `application/json` | Browsers, `curl`, any HTTP client |
| **gRPC** | HTTP/2, binary Protobuf with framing | `application/grpc` | Native gRPC clients (`agynd`, Terraform provider) |
| **gRPC-Web** | HTTP/1.1 or HTTP/2, binary with modified framing | `application/grpc-web` | Browser clients that need streaming |

ConnectRPC handlers are standard Go `http.Handler` — they compose with existing middleware (CORS, auth, request ID, recovery) and work with OpenZiti's `net.Listener` (which implements Go's standard `net.Listener` interface).

### Why ConnectRPC

- **Single handler, multiple protocols.** The Gateway does not need protocol-sniffing code or separate gRPC and HTTP servers. Content-Type routing is built into ConnectRPC.
- **Proto as single source of truth.** The external API is defined by proto service definitions in `agynio/api`. No separate OpenAPI specs to maintain.
- **Buf ecosystem alignment.** The project already uses Buf for proto linting, breaking change detection, and the BSR. ConnectRPC code generation (`protoc-gen-connect-go`) integrates with `buf generate`.
- **Browser-friendly without translation.** The Connect protocol serves JSON over HTTP/1.1 — browsers and `curl` can call the API directly without a REST-to-gRPC translation layer.
- **Native gRPC for agents.** `agynd` and other Go clients call the same endpoint using standard gRPC — no JSON serialization overhead.

## External API Surface

The Gateway defines its own **proto services** in `agynio/api` that describe the externally-exposed methods. These gateway proto services import and reuse message types from internal service protos — no duplication of request/response schemas.

```proto
// Example: agynio/api/gateway/v1/threads.proto
syntax = "proto3";
package agynio.api.gateway.v1;

import "agynio/api/threads/v1/threads.proto";

service ThreadsGateway {
  rpc GetMessages(agynio.api.threads.v1.GetMessagesRequest)
      returns (agynio.api.threads.v1.GetMessagesResponse);
  rpc SendMessage(agynio.api.threads.v1.SendMessageRequest)
      returns (agynio.api.threads.v1.SendMessageResponse);
}
```

Only methods intended for external use appear in gateway proto services. Internal-only methods (e.g., `RegisterIdentity`) are not listed — no handler means no access.

### Exposed Services

| Gateway Proto Service | Internal Service | Methods |
|-----------------------|-----------------|---------|
| `AgentsGateway` | [Agents](agents-service.md) | All CRUD methods for agents and sub-resources |
| `ThreadsGateway` | [Threads](threads.md) | All methods |
| `ChatGateway` | [Chat](chat.md) | All methods |
| `NotificationsGateway` | [Notifications](notifications.md) | Subscribe (server-streaming) |
| `FilesGateway` | [Files](media.md) | UploadFile (client-streaming), GetFileMetadata, GetDownloadURL |
| `AgentStateGateway` | [Agent State](agent/state.md) | All methods |
| `TokenCountingGateway` | [Token Counting](token-counting.md) | All methods |
| `LLMGateway` | [LLM](llm.md) | CreateChatCompletion (proxied LLM calls) |
| `TracingGateway` | [Tracing](tracing.md) | Ingest, Query |
| `SecretsGateway` | [Secrets](secrets.md) | ResolveSecretValue |

### Handler Implementation

Each ConnectRPC handler receives a typed proto message (regardless of wire protocol), performs authentication and authorization, calls the internal gRPC service, and returns the response:

```go
func (g *Gateway) GetMessages(
    ctx context.Context,
    req *connect.Request[threadsv1.GetMessagesRequest],
) (*connect.Response[threadsv1.GetMessagesResponse], error) {
    // Authentication, authorization
    // Call internal Threads service via standard gRPC client
    resp, err := g.threadsClient.GetMessages(ctx, req.Msg)
    if err != nil {
        return nil, err
    }
    return connect.NewResponse(resp), nil
}
```

## Classification

The Gateway is a **data plane** service — it carries live API traffic.

## Implementation

The gateway (`agynio/gateway`, Go) uses:
- `connectrpc/connect-go` for handler generation and protocol serving.
- `protoc-gen-connect-go` for server stub generation from gateway proto definitions.
- Standard gRPC clients for calling internal services.

### Ingress Routing

The gateway receives traffic through two Istio VirtualService routes (defined in `agynio/bootstrap`, `stacks/platform/main.tf`):

1. **Subdomain route** (`virtualservice_gateway`): `gateway.agyn.dev/*` → `gateway-gateway:8080`. No URI rewrite.
2. **Path-based route** (`virtualservice_platform_ui`): `agyn.dev/api/*` → `gateway-gateway:8080` with URI rewrite (`/api/` → `/`). This route is defined on the same VirtualService as the platform-ui route.

The UI uses the path-based route (`/api/`) so that the gateway API is served from the same origin as the web app, avoiding CORS overhead.
