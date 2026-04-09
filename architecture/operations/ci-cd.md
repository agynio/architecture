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

Services with in-cluster E2E suites add a dedicated `e2e` job to `ci.yml`. The job provisions a k3d-backed bootstrap cluster, installs tooling, and runs DevSpace tests against the cluster.

**Job definition:**

- `runs-on: ubuntu-latest`
- `timeout-minutes: 60`
- `permissions`: `contents: read`, `packages: write`

**Disk space reclamation:** remove preinstalled toolchains to make room for the k3d cluster (Java/JDKs, .NET SDKs, Swift toolchain, Haskell/GHC, Julia, Android SDKs).

**PR image build + push:**

1. Set `PR_IMAGE=ghcr.io/agynio/<service>:pr-${{ github.event.pull_request.number || github.run_id }}-${{ github.sha }}` in the job environment.
2. Build and push the image with Docker Buildx after logging into GHCR.

**Bootstrap checkout:** checkout `agynio/bootstrap` at `main` into `bootstrap/`.

**Tool installation:**

- kubectl v1.28.7
- k3d v5.7.5
- Terraform 1.6.6
- DevSpace v6.3.20

**Cluster provisioning:** run `bootstrap/apply.sh -y`. The script applies Terraform stacks in order (`k8s`, `system`, `routing`, `deps`, `ziti`, `data`, `platform`, `apps`) with a CA install step between `system` and `routing`. The kubeconfig is written to `bootstrap/stacks/k8s/.kube/agyn-local-kubeconfig.yaml`.

**Platform health verification:** run `bootstrap/.github/scripts/verify_platform_health.sh` to poll ArgoCD applications, check pod and job health, and verify Ziti overlay readiness.

**E2E execution:** run `devspace run-pipeline test-e2e` with `KUBECONFIG` pointing at the bootstrap kubeconfig and `PR_IMAGE` set to the PR tag. Additional test-suite variables (for example, Argos metadata for Playwright) are passed as needed.

**Service patching modes:**

- **Source-sync (Go services):** CI runs `devspace dev` first to patch the service deployment with a dev container and synced source. `devspace run-pipeline test-e2e` runs afterward against the patched pod.
- **Image-swap (frontends/built artifacts):** CI builds and pushes the PR image. The `test-e2e` pipeline swaps the deployment image via `kubectl set image` using `PR_IMAGE` before running tests.

**Artifacts:** upload Playwright reports and traces with `actions/upload-artifact` when applicable (for example, always upload `playwright-report/`, upload `test-results/` on failures).

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
