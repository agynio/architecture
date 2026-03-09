# Local Development

## Bootstrap

The single source of truth for running the full Agyn cluster locally is:

**Repository:** [`agynio/bootstrap_v2`](https://github.com/agynio/bootstrap_v2)

No other custom docker-compose files or local setups should exist. All services are configured and deployed from this repo.

### How It Works

Bootstrap uses **Terraform** to provision a local Kubernetes cluster and deploy all services via **Argo CD**:

| Stack | Purpose |
|-------|---------|
| `stacks/k8s` | Creates a local K8s cluster (k3d) |
| `stacks/system` | Deploys system infrastructure (Istio, Argo CD, cert-manager, namespaces) |
| `stacks/routing` | Configures ingress, TLS, and DNS routing |
| `stacks/platform` | Deploys all Agyn services as Argo CD Applications |

### Quick Start

```bash
git clone https://github.com/agynio/bootstrap_v2.git
cd bootstrap_v2
chmod +x apply.sh
./apply.sh        # interactive
./apply.sh -y     # auto-approve
```

### Requirements

- Terraform ≥ 1.5.0
- kubectl (for kubeconfig merge)
- Docker

### Local Endpoints

| Service | URL |
|---------|-----|
| Platform UI | `https://agyn.dev:2496/` |
| Platform API | `https://agyn.dev:2496/api` |
| Argo CD | `https://argocd.agyn.dev:2496/` |
| LiteLLM | `https://litellm.agyn.dev:2496/` |
| Vault | `https://vault.agyn.dev:2496/` |

### Services Deployed

The platform stack deploys all services as Argo CD Applications pointing to Helm charts from `oci://ghcr.io/agynio/charts`:

- platform-server
- platform-ui
- docker-runner
- agent-state
- notifications
- gateway
- NCPS

Each Application tracks a chart version and image tag configurable via Terraform variables.

## Service Development with DevSpace

Individual service development uses **DevSpace** against the local cluster provisioned by bootstrap.

### How It Works

1. Bootstrap provisions the full cluster with all services running from released images.
2. DevSpace attaches to a running service's pod, syncs local source code, and restarts the process with hot-reload.
3. The Argo CD Application's auto-sync is paused during the dev session and restored after.

### Example: Platform Server

```bash
cd platform/packages/platform-server
devspace dev
```

This:
- Pauses Argo CD auto-sync for the platform-server Application.
- Syncs local source into the running container.
- Starts the dev server with hot-reload.
- Restores auto-sync on session exit.

### Principle

- **Bootstrap** owns cluster topology and service versions.
- **DevSpace** owns the developer inner loop (code → sync → test).
- No per-service docker-compose or standalone local environments.
