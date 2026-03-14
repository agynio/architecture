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
| [Implementation](#implementation) | Service repo with application code, Dockerfile, Helm chart, DevSpace config |
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
├── devspace.yaml          # DevSpace config: dev mode + E2E tests
├── Dockerfile
├── README.md
├── Makefile
└── go.mod
```

### README

Each service README follows a standard structure:

~~~markdown
# <Service Name>

Short description of what the service does.

Architecture: [<Service Name>](https://github.com/agynio/architecture/blob/main/architecture/<service>.md)

## Local Development

Full setup: [Local Development](https://github.com/agynio/architecture/blob/main/architecture/operations/local-development.md)

### Prepare environment

```bash
git clone https://github.com/agynio/bootstrap.git
cd bootstrap
chmod +x apply.sh
./apply.sh -y
```

See [bootstrap](https://github.com/agynio/bootstrap) for details.

### Run from sources

```bash
# Deploy once (exit when healthy)
devspace dev

# Watch mode (streams logs, re-syncs on changes)
devspace dev -w
```

### Run tests

```bash
devspace run test:e2e
```

See [E2E Testing](https://github.com/agynio/architecture/blob/main/architecture/operations/e2e-testing.md).
~~~

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


### DevSpace

Each service provides a `devspace.yaml` with three commands:

| Command | Purpose |
|---------|---------|
| `devspace dev` | Deploy once: patch the service pod with a dev container, sync source, start the service, exit when healthy. Used by CI and scripts. |
| `devspace dev -w` | Watch mode: same as `devspace dev` but keeps running, streams logs, and re-syncs on file changes. Used during local development. |
| `devspace run test:e2e` | Run E2E tests in a separate test pod inside the cluster. Self-contained: deploys test pod → syncs source → runs tests → cleans up. |

`devspace dev` and `devspace dev -w` patch the existing service deployment with a dev container image, replacing the released image with source-based hot-reload. The `-w` flag is implemented via a pipeline flag (see gateway's `devspace.yaml` for the pattern).

`devspace run test:e2e` does not touch the service pod. It deploys a separate test pod, syncs test code into it, and executes the tests. See [E2E Testing](e2e-testing.md).

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
| Push `v*.*.*` tag | `<semver>`, `latest`, `sha-<commit>` |

### Helm Chart Publishing

On `v*.*.*` tag push:

1. Package chart with version extracted from tag.
2. Push to `oci://ghcr.io/agynio/charts`.

---

## Bootstrap

Register the service in `agynio/bootstrap_v2` so it is deployed in the local cluster. See [Local Development](local-development.md) for how bootstrap provisions the cluster.

---

## E2E Tests

Each service includes end-to-end tests that verify behavior against a running environment provisioned by bootstrap. Tests run **inside the cluster** in a dedicated test pod via DevSpace — the service pods run their pinned release images untouched. See [E2E Testing](e2e-testing.md) for the full methodology.

### Structure

- Tests live in `test/e2e/` within the service repo.
- Tests connect to services via Kubernetes DNS (e.g., `teams:50051`, `threads:50051`).
- A separate test pod is deployed by DevSpace using `component-chart`. The dev container image depends on the test language (Go, Node, Playwright, etc.).

### DevSpace Setup

Add E2E sections to `devspace.yaml`: a `deployments.e2e-runner` (component-chart), a `dev.e2e-runner` (sync), and a `pipelines.test:e2e` (deploy → sync → exec → cleanup). The user-facing command is `devspace run test:e2e`. Follow the pattern in [E2E Testing](e2e-testing.md).

### E2E Tests in CI

The CI workflow provisions the environment using bootstrap and runs `devspace run test:e2e` inside the cluster. No custom docker-compose or Kind-based setups — bootstrap is the single source of truth for the test environment.
