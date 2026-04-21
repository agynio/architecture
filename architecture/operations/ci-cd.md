# CI/CD

Every service follows the same CI/CD pattern: GitHub Actions publish a container image and a Helm chart to GHCR on every release.

## Artifacts

| Artifact | Registry | Tag pattern |
|----------|----------|-------------|
| Container image | `ghcr.io/agynio/<service>` | `sha-<short>`, `<semver>`, `latest` |
| Helm chart | `oci://ghcr.io/agynio/charts/<service>` | `<semver>` (from tag `v*.*.*`) |

## Workflows

Each service repository contains GitHub Actions workflows under `.github/workflows/`:

### Image Build (`docker-ghcr.yml` or combined `release.yml`)

**Trigger:** Push of a `v*.*.*` tag.

**Steps:**
1. Checkout repository.
2. Set up Docker Buildx with multi-platform support. Build targets: `linux/amd64`, `linux/arm64`.
3. Log in to GHCR using `GITHUB_TOKEN`.
4. Build and push image with metadata-driven tags:
   - `sha-<short>` — every release.
   - `<major>.<minor>.<patch>`, `<major>.<minor>`, `<major>` — semver tags.
   - `latest` — stable semver tags (no pre-release suffix).
5. Layer caching via GitHub Actions cache (`type=gha`).
6. Verify the pushed manifest includes `linux/amd64` and `linux/arm64`:
   `docker buildx imagetools inspect ghcr.io/agynio/<service>:<tag>`.

#### Multi-Architecture Image Requirements

- **Base images** — Use only official multi-arch base images. Verify that the chosen tag publishes manifests for both `amd64` and `arm64`.
- **No architecture-specific artifacts** — Do not download pre-built binaries by hard-coded architecture. Use `ARG TARGETARCH` and select architecture at build time.
- **Go services** — Use `CGO_ENABLED=0` for static linking. Set `GOARCH=$TARGETARCH` explicitly when using `FROM --platform=$BUILDPLATFORM` — this enables native-speed cross-compilation without QEMU emulation.
- **Multi-stage builds** — Build stage uses `FROM --platform=$BUILDPLATFORM`; runtime stage uses the default (target) platform.

Example Dockerfile:

```Dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM golang:1.22-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
ARG TARGETOS TARGETARCH
ENV CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH
RUN go build -o /out/service ./cmd/service

FROM alpine:3.19
WORKDIR /app
COPY --from=build /out/service /app/service
ENTRYPOINT ["/app/service"]
```

### Helm Release (`helm-release.yml` or combined `release.yml`)

**Trigger:** Push of a `v*.*.*` tag.

**Steps:**
1. Checkout repository.
2. Extract version from tag (strip `v` prefix).
3. Set up Helm.
4. Build chart dependencies (`helm dependency build`).
5. Lint chart.
6. Package chart with extracted version.
7. Push to `oci://ghcr.io/agynio/charts`.

### CI (`ci.yml`)

**Trigger:** Pull requests and push to `main`.

**Steps:** Lint, test, build — service-specific.

### E2E Job

Every service's `ci.yml` includes an e2e job that composes two composite actions — one from [`agynio/bootstrap`](https://github.com/agynio/bootstrap) to provision the cluster, one from [`agynio/e2e`](https://github.com/agynio/e2e) to run the tests — with a service-specific deploy step in between:

```yaml
jobs:
  e2e:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: agynio/bootstrap/.github/actions/provision@main
      - name: Deploy this service from source
        run: devspace dev
      - uses: agynio/e2e/.github/actions/run-tests@main
        with:
          service: <service-name>
```

Three logical blocks: cluster provisioning (owned by `agynio/bootstrap`), service deployment from source (owned by the service — typically `devspace dev`, but a service with custom bring-up needs does its own thing here), and test execution (owned by `agynio/e2e`). Cluster-level concerns (kubectl/k3d/Terraform versions, stack order, verification) live in `agynio/bootstrap` and propagate via the `provision` action. E2E-level concerns (suite discovery, test selection, pod lifecycle, artifacts) live in `agynio/e2e` and propagate via the `run-tests` action. No image is built or pushed.

See [E2E Testing — CI Integration](e2e-testing.md#ci-integration) for the full action definitions and rationale for the two-action split.

## Base Helm Chart

All service Helm charts inherit from the shared library chart:

```yaml
# charts/<service>/Chart.yaml
dependencies:
  - name: service-base
    version: ">=0.1.4 <1.0.0"
    repository: oci://ghcr.io/agynio/charts
```

The base chart (`agynio/base-chart`, published as `service-base`) provides common templates for Deployment, Service, ServiceAccount, HPA, and Ingress. Service charts override values without duplicating boilerplate.

Source: [`agynio/base-chart`](https://github.com/agynio/base-chart)

## Adding a New Service

1. Create the service repo under `agynio/`.
2. Add a `Dockerfile`.
3. Add a Helm chart in `charts/<service>/` with a dependency on `service-base`.
4. Add CI/CD workflows following the patterns above.
5. Register the chart in the bootstrap configuration (see [Local Development](local-development.md)).
