# Runner

## Overview

The Runner executes workloads (agent containers, workspace containers, sidecars). Multiple implementations exist for different backends:

| Implementation | Backend | Status |
|----------------|---------|--------|
| `docker-runner` | Docker Engine | Existing (`agynio/platform`) |
| `k8s-runner` | Kubernetes | Planned |

## gRPC API

Defined in `agynio/api` at `proto/agynio/api/runner/v1/runner.proto`.

### Workload Lifecycle

| RPC | Description |
|-----|-------------|
| `StartWorkload` | Start a workload (main container + optional sidecars with shared network) |
| `StopWorkload` | Stop a running workload |
| `RemoveWorkload` | Remove a workload and optionally its volumes |
| `InspectWorkload` | Inspect workload state (id, image, labels, mounts, status) |
| `TouchWorkload` | Update last-used timestamp (TTL keepalive) |

### Query

| RPC | Description |
|-----|-------------|
| `GetWorkloadLabels` | Get labels for a workload |
| `FindWorkloadsByLabels` | Find workloads matching a label set |
| `ListWorkloadsByVolume` | List workloads using a specific volume |

### Execution

| RPC | Description |
|-----|-------------|
| `Exec` | Bidirectional streaming exec (replaces HTTP+WS exec) |
| `CancelExecution` | Cancel a running execution |

Exec supports:
- Interactive (TTY) and non-interactive modes.
- Wall timeout, idle timeout, kill-on-timeout.
- Stdin streaming, stdout/stderr separation.
- Exit code, reason (completed, timeout, idle_timeout, cancelled, error).

### Streaming

| RPC | Description |
|-----|-------------|
| `StreamWorkloadLogs` | Server-streaming log output (follow mode supported) |
| `StreamEvents` | Server-streaming runtime events |

### Storage

| RPC | Description |
|-----|-------------|
| `PutArchive` | Upload a tar archive into a workload filesystem |
| `RemoveVolume` | Remove a named volume |

## Workload Model

A workload consists of:
- **Main container** — The primary process.
- **Sidecars** — Optional containers sharing the same network namespace as the main container.
- **Volumes** — Ephemeral or named (persistent) volumes mounted into containers.

## Authentication

The docker-runner uses HMAC-based authentication. Every gRPC request includes metadata derived from a shared secret (`DOCKER_RUNNER_SHARED_SECRET`).

## Control/Data Plane Split

The Runner currently combines scheduling (control) and execution (data) concerns. Moving forward it should be split:

- **Runner Controller** (control plane): Manages workload desired state, reconciles container lifecycle, handles TTL and cleanup.
- **Runner Worker** (data plane): Provides exec, log streaming, archive, and compute surface for running workloads.

See [Control Plane & Data Plane](control-data-plane.md) and [Open Questions](open-questions.md#runner-split).
