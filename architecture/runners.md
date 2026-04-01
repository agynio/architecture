# Runners

## Overview

The Runners service manages runner registrations and workload runtime state. It is the central registry for:

1. **Runners** — registered runner instances (cluster-scoped and org-scoped), their enrollment state, and metadata.
2. **Workloads** — the runtime state of workloads running on registered runners. Which workloads are running, on which runner, with which containers.

The [Agents Orchestrator](agents-orchestrator.md) reads and writes workload state through this service. The [Gateway](gateway.md) exposes query methods for the UI. The [Terminal Proxy](terminal-proxy.md) resolves which runner hosts a workload to route exec connections.

## API

### Runner Management

| Method | Description |
|--------|-------------|
| **RegisterRunner** | Register a new runner. Creates the runner record, creates a per-runner OpenZiti service via [Ziti Management](openziti.md), registers an identity (type `runner`) in [Identity](identity.md), writes authorization tuples, and generates a service token |
| **GetRunner** | Get a runner by ID |
| **ListRunners** | List registered runners. Supports filtering by organization |
| **UpdateRunner** | Update a runner's mutable fields (name, labels) |
| **DeleteRunner** | Delete a runner registration. Deletes the per-runner OpenZiti service and revokes the runner's OpenZiti identity via Ziti Management |

### Workload State

| Method | Description |
|--------|-------------|
| **CreateWorkload** | Record a new running workload (runner ID, workload ID, thread ID, agent ID, containers, status) |
| **UpdateWorkloadStatus** | Update workload status and container states |
| **DeleteWorkload** | Remove a workload record |
| **GetWorkload** | Get a workload by ID. Returns workload details including runner ID and containers |
| **ListWorkloadsByThread** | List workloads for a thread. Used by the UI to populate the container popover |

## Runner Resource

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique runner identifier |
| `name` | string | Display name |
| `organization_id` | string (UUID), nullable | Organization scope. Null for cluster-scoped runners |
| `labels` | map<string, string> | Key-value labels describing runner capabilities (e.g., `gpu: "true"`, `region: "eu-west-1"`). Used for [runner selection](#runner-selection). Set at registration time, mutable via `UpdateRunner` |
| `identity_id` | string (UUID) | Runner's identity in the [Identity](identity.md) service |
| `service_token_hash` | string | SHA-256 hash of the service token |
| `openziti_service_name` | string | Per-runner OpenZiti service name (`runner-{id}`) |
| `status` | enum | `pending`, `enrolled`, `offline` |
| `created_at` | timestamp | Creation time |
| `updated_at` | timestamp | Last modification time |

### Organization Scoping

Cluster-scoped runners (`organization_id: null`) are available to all organizations. Org-scoped runners are available only to the owning organization. See [Runner Selection](#runner-selection) for how the orchestrator picks a runner.

## Runner Selection

The [Agents Orchestrator](agents-orchestrator.md) selects a runner for each agent workload using organization scoping and label matching:

1. **Scope filtering** — collect eligible runners: org-scoped runners matching the agent's `organization_id`, plus all cluster-scoped runners. Only runners with status `enrolled` are eligible.
2. **Label matching** — if the agent defines `runner_labels` (see [Agent — runner_labels](resource-definitions.md#agent)), filter eligible runners to those whose `labels` contain all key-value pairs from the agent's `runner_labels`. Exact string equality on both key and value. A runner may have additional labels beyond what the agent requires.
3. **Random selection** — from the filtered set, pick one runner at random.

If the agent defines no `runner_labels`, step 2 is skipped — any eligible runner matches. If no runners remain after filtering, the workload fails to schedule.

## Workload Resource

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Workload ID (matches the ID on the Runner) |
| `runner_id` | string (UUID) | Runner hosting this workload |
| `thread_id` | string (UUID) | Thread this workload serves |
| `agent_id` | string (UUID) | Agent this workload runs |
| `organization_id` | string (UUID) | Organization scope (denormalized from agent) |
| `status` | enum | `starting`, `running`, `stopping`, `stopped`, `failed` |
| `containers` | list | Containers in the workload (see below) |
| `created_at` | timestamp | Creation time |
| `updated_at` | timestamp | Last status update |

### Container

| Field | Type | Description |
|-------|------|-------------|
| `container_id` | string | Container identifier within the workload |
| `name` | string | Display name |
| `role` | enum | `main`, `sidecar`, `init` |
| `image` | string | Container image |
| `status` | enum | `running`, `terminated`, `waiting` |

## Registration Flow

```mermaid
sequenceDiagram
    participant Admin as Admin
    participant RS as Runners Service
    participant ZM as Ziti Management
    participant I as Identity
    participant Auth as Authorization

    Admin->>RS: RegisterRunner(name, organization_id?, labels?)
    RS->>ZM: Create OpenZiti service "runner-{id}" (roleAttributes: ["runner-services"])
    RS->>I: RegisterIdentity(id, type: runner)
    RS->>Auth: Write(identity:runnerId, runner:bind, org/cluster scope)
    RS->>RS: Generate service token, store runner record
    RS-->>Admin: Runner record + service token
```

1. Admin calls `RegisterRunner` (via `agyn` CLI or Terraform).
2. Runners service creates a per-runner OpenZiti service `runner-{id}` with `roleAttributes: ["runner-services"]` via [Ziti Management](openziti.md). This service is what callers will dial to reach this specific runner.
3. Runners service registers the runner's identity in the [Identity](identity.md) service with `identity_type: runner`.
4. Runners service writes authorization tuples granting the runner its permissions.
5. Runners service generates a service token, stores the runner record (including `openziti_service_name`), and returns the token.
6. The service token is provided to the runner deployment.

Static OpenZiti policies handle access — `runners-bind` allows identities with `#runners` to bind `#runner-services`, and `orchestrators-dial-runners` allows identities with `#orchestrators` to dial `#runner-services`. No per-runner policy creation is needed. See [OpenZiti Integration — Service Policies](openziti.md#static-policies).

Cluster-scoped runners are registered by the cluster admin. Org-scoped runners are registered by an organization admin.

## Enrollment

When a runner starts, it presents the service token to the platform enrollment endpoint. The platform validates the token against the Runners service, creates an OpenZiti identity via [Ziti Management](openziti.md), enrolls it, and returns the enrolled identity (certificate + key) along with the service name (`runner-{runnerId}`). See [OpenZiti Integration — Runner Provisioning](openziti.md#runner-provisioning) for the full enrollment sequence.

After enrollment, the runner binds its per-runner OpenZiti service (`runner-{runnerId}`) and begins accepting workload commands from the Orchestrator.

The service token is long-lived and reusable. If the runner restarts, it re-enrolls with the same token and receives a new OpenZiti identity. The previous identity is cleaned up by Ziti Management lease GC. All runners — whether deployed as platform infrastructure or by an enterprise admin — follow this same flow.

## Deletion

`DeleteRunner` cleans up all associated resources:

1. Deletes the per-runner OpenZiti service (`runner-{runnerId}`) via Ziti Management.
2. Revokes the runner's current OpenZiti identity via Ziti Management.
3. Removes authorization tuples.
4. Removes the runner record from PostgreSQL.

## Workload State Management

The [Agents Orchestrator](agents-orchestrator.md) is the sole writer of workload state. The orchestrator calls the Runners service to record workload lifecycle events:

1. **Start**: orchestrator starts a workload on a runner via Runner `StartWorkload`, then calls `CreateWorkload` on the Runners service with the runner ID, workload ID, thread ID, agent ID, and initial container list.
2. **Update**: orchestrator detects status changes during reconciliation (via Runner `InspectWorkload`) and calls `UpdateWorkloadStatus`.
3. **Stop**: orchestrator stops a workload via Runner `StopWorkload`, then calls `DeleteWorkload`.

The Runners service is a passive store — it does not interact with runners directly. It records what the orchestrator tells it.

## Gateway Exposure

The following methods are exposed through the [Gateway](gateway.md) for the UI:

| Gateway Service | Methods |
|----------------|---------|
| `RunnersGateway` | `ListWorkloadsByThread`, `GetWorkload` |

The UI calls `ListWorkloadsByThread` to populate the container popover in the conversation header. It calls `GetWorkload` to get container details before opening a terminal session.

## Terminal Proxy Integration

The [Terminal Proxy](terminal-proxy.md) needs to reach the specific runner hosting a workload. The flow:

1. UI calls `GetWorkload` (via Gateway) to get workload details including `runner_id`.
2. UI opens a WebSocket to the Terminal Proxy with `workloadId` and `containerId`.
3. Terminal Proxy calls `GetWorkload` on the Runners service to resolve `runner_id`.
4. Terminal Proxy dials the specific runner via OpenZiti: `zitiContext.Dial("runner-{runnerId}")`.

Per-runner OpenZiti addressing is established at registration time — each runner has its own OpenZiti service. See [OpenZiti Integration — Runner Provisioning](openziti.md#runner-provisioning).

## Data Store

PostgreSQL. The Runners service owns its database with `runners` and `workloads` tables.

## Classification

| Aspect | Detail |
|--------|--------|
| **Plane** | Mixed — control (registration) + data (workload state queries) |
| **API** | gRPC (internal) + Gateway (external via ConnectRPC) |
| **State** | PostgreSQL |
