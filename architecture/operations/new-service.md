# New Service Development

This document describes the process for developing a new service from API schema to production deployment.

## Steps

```mermaid
flowchart LR
    A[API Schema] --> B[Implementation]
    B --> C[CI/CD]
    C --> D[Bootstrap]
    D --> E[E2E Tests]
```

| Step | Outcome |
|------|---------|
| [API Schema](#api-schema) | Proto and/or OpenAPI definitions merged in `agynio/api` |
| [Implementation](#implementation) | Service repo with application code, Dockerfile, Helm chart |
| [CI/CD](#cicd) | GitHub Actions publish image and chart to GHCR on every release |
| [Bootstrap](#bootstrap) | Service deployed in the local cluster via Argo CD |
| [E2E Tests](#e2e-tests) | Automated tests verify the service in a real cluster |

---

## API Schema

All API schemas live in `agynio/api`. The service repo does not contain schema definitions.

### Internal API (gRPC)

Add proto definitions under `proto/agynio/api/<service>/v1/`:

| Aspect | Convention |
|--------|-----------|
| Package | `agynio.api.<service>.v1` |
| Go package option | `github.com/agynio/api/gen/agynio/api/<service>/v1;<service>v1` |
| Linting | Buf `STANDARD` rules |
| Breaking change detection | Buf `FILE` policy |

Proto is published to `buf.build/agynio/api` via the existing `buf-publish` workflow in `agynio/api`.

### External API (OpenAPI)

Add an OpenAPI spec under `openapi/<service>/v1/`:

| Aspect | Convention |
|--------|-----------|
| Spec version | OpenAPI 3.0.3 |
| Structure | Modular — one YAML per component, referenced via `$ref` |
| Linting | Spectral (`.spectral.yaml` in `agynio/api`) |
| Pagination | Cursor-based on all list endpoints |
| Errors | `Problem` schema (RFC 7807) |

OpenAPI spec is published to GHCR via the existing `openapi-publish` workflow in `agynio/api`.

### Workflow

1. Create a PR in `agynio/api` with the new proto and/or OpenAPI definitions.
2. CI runs Buf lint + breaking change detection (proto) and Spectral lint (OpenAPI).
3. Merge. Buf publish pushes the updated module; OpenAPI publish pushes the spec artifact.

---

## Implementation

Create a new repo under `agynio/<service>`.

### Repository Structure

```
agynio/<service>/
├── .github/workflows/     # CI + release workflows
├── charts/<service>/      # Helm chart
│   ├── Chart.yaml         # Depends on service-base
│   ├── values.yaml
│   └── templates/
├── cmd/<service>/
│   └── main.go            # Entrypoint
├── internal/              # Application code
├── test/
│   └── e2e/               # E2E tests
├── buf.gen.yaml           # Proto code generation config
├── Dockerfile
├── Makefile
└── go.mod
```

### Proto Code Generation

The service generates Go code from `agynio/api` protos locally using `buf generate` with a `buf.gen.yaml` pointing at `buf.build/agynio/api`. Generated code is written to an internal `.gen/` directory. It is not committed — generated at build time (in Dockerfile and CI).

### Helm Chart

The chart inherits from the shared base chart:

```yaml
# charts/<service>/Chart.yaml
dependencies:
  - name: service-base
    version: ">=0.1.4 <1.0.0"
    repository: oci://ghcr.io/agynio/charts
```

The base chart (`agynio/base-chart`) provides templates for Deployment, Service, ServiceAccount, HPA, and Ingress. Service charts override values.

---

## CI/CD

Add GitHub Actions workflows under `.github/workflows/` in the service repo. All services follow the same pattern (see [CI/CD](ci-cd.md)).

### Workflows

| Workflow | Trigger | Artifacts |
|----------|---------|-----------|
| `ci.yml` | Pull requests | Lint, test, build |
| `release.yml` | Push to `main` or `v*.*.*` tag | Container image + Helm chart to GHCR |

### Image Tags

| Condition | Tags |
|-----------|------|
| Push to `main` | `edge` |
| Push `v*.*.*` tag | `<semver>`, `latest`, `sha-<commit>` |

### Helm Chart Publishing

On `v*.*.*` tag push:

1. Package chart with version extracted from tag.
2. Push to `oci://ghcr.io/agynio/charts`.

---

## Bootstrap

Register the service in `agynio/bootstrap_v2` so it is deployed in the local cluster.

### Steps

1. Add Terraform variables for the service's chart version, image tag overrides, and configuration values (DB passwords, endpoints, etc.).
2. Add a `locals` block that builds the Helm values for the service (image, config, resource references).
3. Add an `argocd_application` resource that deploys the chart from GHCR into the `platform` namespace with the appropriate sync wave.
4. If the service requires a database, add a `kubernetes_stateful_set_v1` for the database (PostgreSQL) with its PVC, and wire the connection string into the service values.
5. If the service requires an Istio VirtualService (externally reachable), add the routing manifest.

### Argo CD Application Pattern

```hcl
resource "argocd_application" "<service>" {
  metadata {
    name      = "<service>"
    namespace = "argocd"
    annotations = {
      "argocd.argoproj.io/sync-wave" = "<wave>"
    }
  }

  spec {
    project = "default"

    source {
      repo_url        = local.platform_chart_repo_host
      chart           = "agynio/charts/<service>"
      target_revision = var.<service>_chart_version

      helm {
        values = local.<service>_values
      }
    }

    destination {
      server    = var.destination_server
      namespace = var.platform_namespace
    }

    sync_policy {
      # automated sync based on bootstrap-level toggles
    }
  }
}
```

### DevSpace

Add a `devspace.yaml` to the service repo for inner-loop development against the bootstrap cluster (see [Local Development](local-development.md)).

---

## E2E Tests

Each service includes automated end-to-end tests that verify behavior in a real cluster with real dependencies.

### In-Repo E2E Tests

Located in `test/e2e/` within the service repo. These tests start the gRPC server in-process against a real database (PostgreSQL via testcontainers or a dedicated test instance), exercise the full request path through the gRPC API, and verify responses.

Pattern (Go):

1. Spin up the database (testcontainers or connect to a running instance).
2. Apply migrations.
3. Start the gRPC server on a random port.
4. Create a gRPC client.
5. Run test cases: create, get, list, update, delete resources; verify responses and error codes.

### Smoke Tests in CI (Kind)

For services with external dependencies (Redis, object storage, etc.), a Kind-based smoke test workflow runs in CI:

1. Build the container image.
2. Create a Kind cluster.
3. Load the image into Kind.
4. Install dependencies via Helm (Redis, MinIO, etc.).
5. Install the service chart with the test image.
6. Port-forward the service.
7. Run smoke tests from `test/smoke/`.

### What to Test

| Category | Examples |
|----------|---------|
| CRUD operations | Create, get, list, update, delete for each resource type |
| Reference integrity | Creating a resource with a non-existent foreign reference returns an error |
| Proxy / resolution | For services with proxy endpoints (e.g., LLM proxy), verify request forwarding and response passthrough |
| Error cases | Invalid input, not found, duplicate creation |
