# Local Development

## Bootstrap

The single source of truth for running the full Agyn cluster locally is:

**Repository:** [`agynio/bootstrap_v2`](https://github.com/agynio/bootstrap_v2)

No other custom docker-compose files or local setups should exist. All services are configured and deployed from this repo.

Bootstrap uses **Terraform** to provision a local Kubernetes cluster (k3d) and deploy all services via **Argo CD**. Setup instructions, stack descriptions, and local endpoints are documented in the [bootstrap_v2 repository](https://github.com/agynio/bootstrap_v2).

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
