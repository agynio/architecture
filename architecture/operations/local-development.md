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

### Development Modes

| Mode | Command | Behavior | Why |
|------|---------|----------|-----|
| Default | `devspace dev` | Sets up sync, waits for readiness, then exits while leaving the patched pod running. | Enables scripting and CI workflows that need a ready dev environment without a long-running session. |
| Watch | `devspace dev -w` | Sets up sync and stays attached with logs, port forwarding, and file watching. | Provides the interactive inner loop for local iteration. |

### Design Choices

| Concern | Behavior | Why it matters |
|---------|----------|----------------|
| Argo CD integration | Auto-sync is paused on start and restored on exit. | Prevents Argo CD from reverting the dev patch while keeping the cluster convergent afterwards. |
| Pod patching | DevSpace patches the existing deployment (dev image + empty volume for sync) instead of replacing the pod. | Preserves deployment identity and allows the session to end without rolling back the workload. |
| File sync + hot reload | Local code sync completes first, then the dev container starts the app with hot reload. | Ensures the process boots on local code and supports fast iteration without rebuilds. |

The gateway service (`agynio/gateway`) is the reference implementation for this DevSpace setup.

### Default Mode Exit Flow

Default mode is designed to exit cleanly after the service is verifiably running local code.
After the health check completes, the pipeline calls `stop_dev` so the session can exit without reverting the pod.

| Step | Responsibility | Exit behavior |
|------|----------------|---------------|
| `start_dev` | Blocks until initial sync finishes and the dev container starts. | Guarantees the health check hits the synced code path. |
| Health check | Polls the service health endpoint after sync. | Confirms the dev workload is ready before teardown. |
| `stop_dev` | Deregisters active dev sessions and tears down sync/forwarding. | Allows DevSpace to exit while leaving the patched pod in place. |
| `WaitDev()` | Blocks only while dev sessions are registered. | Exits immediately after `stop_dev` in default mode; remains running in watch mode. |

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
