# Local Development

This document covers **developing the platform itself** — running a local cluster that service code can be hot-reloaded into. To simply **run** the platform locally (use it, demo it, develop agents or apps against it), the same image is all you need — see [Local Bundle](local-bundle.md) and [`agyn local`](../agyn-cli.md#local-platform-commands-agyn-local).

## The Cluster

The full Agyn platform runs locally as the prebuilt VM image, started with [`agyn local start`](../agyn-cli.md#local-platform-commands-agyn-local). `agyn local kubeconfig` adds it to the kubeconfig as its own context, and DevSpace attaches to it from there.

No other custom docker-compose files or local setups should exist.

Nothing inside the VM reconciles: the platform is a Helm release that is applied once, at bake time. A DevSpace patch therefore stays in place until something rewrites the workload — an [in-place upgrade](local-bundle.md#upgrade-model), or `agyn local reset`, which restores workloads from the stored release and is the way back to a clean cluster without losing data.

## Service Development with DevSpace

### How It Works

1. The VM boots with every service running the version the image shipped.
2. DevSpace attaches to a running service's pod, syncs local source code, and restarts the process with hot-reload.
3. The patch survives until the workload is rewritten — by `agyn local reset`, an upgrade, or a VM restart.

### Development Modes

| Mode | Command | Behavior | Why |
|------|---------|----------|-----|
| Default | `devspace dev` | Sets up sync, waits for readiness, then exits while leaving the patched pod running. | Enables scripting and CI workflows that need a ready dev environment without a long-running session. |
| Watch | `devspace dev -w` | Sets up sync and stays attached with logs, port forwarding, and file watching. | Provides the interactive inner loop for local iteration. |

### Design Choices

| Concern | Behavior | Why it matters |
|---------|----------|----------------|
| Pod patching | DevSpace patches the existing deployment (dev image + empty volume for sync) instead of replacing the pod. | Preserves deployment identity and allows the session to end without rolling back the workload. |
| File sync + hot reload | Local code sync completes first, then the dev container starts the app with hot reload. | Ensures the process boots on local code and supports fast iteration without rebuilds. |
| Returning to released state | `agyn local reset [--service NAME]` restores workloads from the `agyn-platform` release; data is untouched. | Gives the dev loop an explicit undo, so a patched cluster is never the only way back. |

The gateway service (`agynio/gateway`) is the reference implementation for this DevSpace setup.

Service `devspace.yaml` files still pause an Argo CD Application before patching and restore it afterwards. Nothing in the VM runs Argo CD, so those steps find no Application and warn; they remain for clusters that do have one.

### Default Mode Exit Flow

Default mode is designed to exit cleanly after the service is verifiably running local code.
After the health check completes, the pipeline calls `stop_dev` so the session can exit without reverting the pod.

| Step | Responsibility | Exit behavior |
|------|----------------|---------------|
| `start_dev` | Blocks until initial sync finishes and the dev container starts. | Guarantees the health check hits the synced code path. |
| Health check | Polls the service health endpoint after sync. | Confirms the dev workload is ready before teardown. |
| `stop_dev` | Deregisters active dev sessions and tears down sync/forwarding. | Allows DevSpace to exit while leaving the patched pod in place. |
| `WaitDev()` | Blocks only while dev sessions are registered. | Exits immediately after `stop_dev` in default mode; remains running in watch mode. |

### Example: Gateway

```bash
cd gateway
devspace dev
```

This:
- Syncs local source into the running container.
- Starts the dev server with hot-reload.
- Leaves the patched pod running when the session exits.

### Principle

- **The platform image** owns cluster topology and service versions.
- **DevSpace** owns the developer inner loop (code → sync → reload).
- No per-service docker-compose or standalone local environments.
