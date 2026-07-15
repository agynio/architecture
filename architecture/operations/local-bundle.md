# Local Bundle

A prebuilt VM image that runs the full Agyn platform locally. Instead of provisioning a local cluster (Docker, k3d, Istio, Argo CD, platform services) from scratch, a user downloads one disk image and boots it: download is ~900 MB, boot-to-working-platform is ~30 seconds.

The consumer-facing interface is the [`agyn local`](../agyn-cli.md#local-platform-commands-agyn-local) command group in `agyn-cli`. This document describes the images themselves: how they are layered, built, published, and what runs inside.

## Two-Image Design

The bundle is built in two layers, each owned by its own repository:

| Image | Repository | Contains | Changes |
|-------|------------|----------|---------|
| **Base** | `agynio/bundle-vm-base` | Ubuntu + single-node k3s + cert-manager + Istio + wildcard TLS + Helm/kubectl/debug tooling | Rarely — infrastructure dependencies only |
| **Platform** | `agynio/bundle-vm` | Inherits the base image and bakes in the full Agyn platform, deployed via the unified `agyn-platform` umbrella chart (`oci://ghcr.io/agynio/charts/agyn-platform`) | Every platform release |

The split keeps the expensive, slow-moving infrastructure layer cached and rebuilds only the platform layer per release.

The base image contains no Agyn service versions and no production, staging, external-service, or developer credentials.

### Platform Bake Process

The platform build boots the base QCOW2 as its input disk and:

1. Applies supporting manifests (namespaces, dev secrets, shared Postgres, Istio routing) and installs the data layer (OpenFGA, MinIO) and the platform as a Helm release of the `agyn-platform` umbrella chart.
2. Waits until every workload is Ready — which guarantees every container image is present in the k3s containerd store.
3. Finalizes the disk clean.

No GitOps tooling is involved at any stage — neither image contains Argo CD, and the bake installs the platform directly. The shipped image boots with the platform already deployed and all images local — no registry pulls, no in-VM reconciliation.

### Upgrade Model

Upgrades are **image replacement, not in-place upgrades**. Nothing inside the VM reconciles or upgrades itself. To move to a new platform version, the consumer downloads the new image and recreates the VM (`agyn local upgrade`).

## Artifacts and Publishing

Each release publishes per architecture (`amd64`, `arm64`) to the CDN:

```
downloads.agyn.cloud/bundle-vm/<version>/<arch>/
├── disk.qcow2.xz        # xz-compressed QCOW2 disk
├── checksums.sha256     # sha256 for every file in the directory
├── metadata.json        # name, version, arch, disk format/size, ingress config
└── lima.yaml            # Lima instance template for this image
```

Version resolution for consumers:

```
downloads.agyn.cloud/bundle-vm/latest.json
```

`latest.json` maps `latest` to a concrete version and is updated by the publish flow. Pinned versions bypass it. Consumers verify every download against `checksums.sha256`.

Base images are build inputs, not consumer artifacts — they are published as OCI artifacts to GHCR (`ghcr.io/agynio/bundle-vm-base:<version>-<arch>`) for the platform build to consume.

## Runtime

The image runs under [Lima](https://lima-vm.io/) (QEMU-backed) using the published `lima.yaml`. One instance per machine, named `agyn` — no multi-instance.

### Networking

| Layer | Behavior |
|-------|----------|
| Inside the VM | The Istio ingress gateway terminates TLS on container port 443, surfaced by the `istio-ingressgateway` NodePort `32443` |
| Host | A Lima port forward maps a host port (default `2496`, configurable) onto the NodePort |
| DNS | `*.agyn.dev` resolves to `127.0.0.1` in public DNS — no `/etc/hosts` editing |

Endpoints are therefore `https://console.agyn.dev:<port>`, `https://agyn.dev:<port>`, etc. The host port is the only externally visible knob; everything inside the VM is fixed.

### TLS and Certificate Authority

The image carries a cert-manager-issued CA that signs the `*.agyn.dev` wildcard certificate, stored in the `istio-gateway/agyn-dev-ca` secret. Consumers extract this CA and install it into the system trust store via [`agyn local ca`](../agyn-cli.md#local-platform-commands-agyn-local) so that browsers and CLIs trust the local endpoints without warnings.

Per-install CA generation (host generates the CA and injects it into the VM, rather than extracting the baked one) is an unresolved decision — see [Open Questions](../../open-questions.md#local-bundle-per-install-ca).

## Relationship to Local Development

The bundle serves **consumers of the platform**: running Agyn locally to use it, demo it, or develop agents and apps against it. **Developing the platform itself** (hot-reloading service code inside a live cluster) remains the domain of [Local Development](local-development.md) — bootstrap provisions a mutable GitOps-managed cluster that DevSpace can attach to, which the immutable bundle image deliberately does not support.
