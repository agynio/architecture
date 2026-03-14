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
2. Set up Docker Buildx (multi-platform: `linux/amd64`, `linux/arm64`).
3. Log in to GHCR using `GITHUB_TOKEN`.
4. Build and push image with metadata-driven tags:
   - `sha-<short>` — every release.
   - `<major>.<minor>.<patch>`, `<major>.<minor>`, `<major>` — semver tags.
   - `latest` — stable semver tags (no pre-release suffix).
5. Layer caching via GitHub Actions cache (`type=gha`).

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

**Trigger:** Pull requests.

**Steps:** Lint, test, build — service-specific.

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
