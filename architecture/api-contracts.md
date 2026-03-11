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
| Runner | `agynio/api/runner/v1/runner.proto` |
| Notifications | `agynio/api/notifications/v1/notifications.proto` |
| Files | `agynio/api/files/v1/files.proto` |
| Agent State | `agynio/api/agent_state/v1/agent_state.proto` |

### Conventions

- Package naming: `agynio.api.<service>.v1`
- Go package: `github.com/agynio/api/gen/agynio/api/<service>/v1;<service>v1`
- Buf lint: `STANDARD`
- Breaking change detection: `FILE`

## External API — OpenAPI

| Aspect | Details |
|--------|---------|
| Protocol | REST (HTTP/1.1 or HTTP/2) |
| Spec | OpenAPI 3.0.3 |
| Registry | GitHub Container Registry (GHCR) |
| Spec path | `openapi/` in `agynio/api` |

### OpenAPI Specs

| API | Spec Path |
|-----|-----------|
| Team Management | `openapi/team/v1/openapi.yaml` |

### Conventions

- Modular schemas: one YAML file per component, referenced via `$ref`.
- Linting: Spectral (`.spectral.yaml`).
- Cursor-based pagination on all list endpoints.
- Error responses: `Problem` schema (RFC 7807 style).
