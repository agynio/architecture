# Gateway

## Overview

The Gateway exposes platform methods for external usage. It is the entry point for external clients (web app, mobile app, integrators) to interact with new platform services.

The gateway is accessible via two ingress paths:

| Path | Host | Prefix stripped | Use case |
|------|------|-----------------|----------|
| Subdomain | `gateway.agyn.dev` | No | Direct access, service-to-service |
| Path-based | `agyn.dev/apiv2/` | Yes (`/apiv2/` → `/`) | UI consumption (same origin as the web app) |

The path-based route allows the web app (platform-ui) to call new gateway-backed APIs without cross-origin requests. Legacy monolith APIs remain at `/api` on the same domain.

## Responsibilities

- Route external API requests to internal services.
- Translate between external (OpenAPI/REST) and internal protocols.
- Stream multipart file uploads to FilesService.UploadFile (client-streaming gRPC).
- Validate requests and responses against OpenAPI specs.
- Authenticate requests and resolve identity + tenant context. See [Authentication](authn.md).

## Classification

The Gateway is a **data plane** service — it carries live API traffic.

## Current Implementation

The gateway (`agynio/gateway`, Go) currently:
- Serves the Team Management API (`/team/v1/`) with OpenAPI request validation (and optional response validation).
- Uses `oapi-codegen` for server stub generation from the OpenAPI spec.
- Proxies remaining routes (`/api/*`, `/health`) to the upstream platform-server.
- Applies CORS, request ID, and recovery middleware.

### Ingress Routing

The gateway receives traffic through two Istio VirtualService routes (defined in `agynio/bootstrap_v2`, `stacks/platform/main.tf`):

1. **Subdomain route** (`virtualservice_gateway`): `gateway.agyn.dev/*` → `gateway-gateway:8080`. No URI rewrite.
2. **Path-based route** (`virtualservice_platform_ui`): `agyn.dev/apiv2/*` → `gateway-gateway:8080` with URI rewrite (`/apiv2/` → `/`). This route is defined on the same VirtualService as the platform-ui and platform-server routes.

The UI uses the path-based route (`/apiv2/`) so that both the legacy monolith API (`/api`) and the new gateway API (`/apiv2/`) are served from the same origin, avoiding CORS overhead.
