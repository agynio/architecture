# Expose Service

## Overview

The Expose service provides **temporary access** to TCP ports inside running agent workloads from external user machines.

It is used for workflows like:

- An agent starts a development server inside its container (e.g., Vite on `5173`).
- The agent requests that port be exposed.
- The platform returns a URL like `http://exposed-<id>.ziti:5173`.
- A user with **Ziti Desktop Edge / ziti tunnel** enrolled can open the URL and reach the dev server.

Expose provisions **OpenZiti services, configs, and policies** dynamically (via [Ziti Management](openziti.md#ziti-management-service)).

**User device enrollment** (minting the enrollment token used by Ziti Desktop Edge) is owned by the [Users](users.md#ziti-user-device-enrollment) service.

## Responsibilities

- **Port shares** — create and revoke port exposures ("shares") for agent workloads.
- **Garbage collection** — reconcile failed provisioning and clean up orphaned shares.
- **Idempotency** — ensure share provisioning and cleanup are safe to retry.

Non-goals:
- Not a general internet ingress.
- Not an HTTP reverse proxy.
- Access scope policy is TBD (see [Open Questions](../open-questions.md#openziti-exposed-ports-access-scope)).

## API

Expose is an internal gRPC service. Selected methods are exposed externally through the [Gateway](gateway.md).

| Method | Caller | Description |
|--------|--------|-------------|
| **AddPort** | Agent | Expose a local TCP port from the agent workload and return a URL (`http://exposed-<id>.ziti:<port>`) |
| **DeletePort** | Agent, User, GC loop | Revoke an exposed port and delete its OpenZiti resources |
| **DeletePortsForAgent** | Agents Orchestrator, GC loop | Revoke all ports for a given agent (workload stop cleanup) |
| **GetPort** | User (Console), internal | Fetch port status and URL |

### AddPort

Creates an exposed port bound to the calling agent.

**Request**

| Field | Type | Description |
|------|------|-------------|
| `thread_id` | string (UUID) | Thread the agent is working in |
| `port` | int32 | Local TCP port inside the agent pod (e.g., `5173`) |
| `description` | string | Optional display text (e.g., `"Vite dev server"`) |

**Response**

| Field | Type | Description |
|------|------|-------------|
| `share_id` | string | 8-character share identifier |
| `url` | string | URL to open (e.g., `"http://exposed-ab12cd34.ziti:5173"`) |
| `status` | enum | `active` |

Expose resolves the calling agent identity to `agent_id` and `organization_id` via `Agents.ResolveAgentIdentity`.

### DeletePort

Revokes an exposed port.

**Request**

| Field | Type | Description |
|------|------|-------------|
| `share_id` | string | Share identifier |

**Response**

Empty.

Deletion is idempotent: deleting an already-deleted port returns success.

### DeletePortsForAgent

Revokes all exposed ports owned by a given agent.

This is primarily used as part of workload stop cleanup: when the Agents Orchestrator stops an agent workload, it calls `DeletePortsForAgent(agent_id)` so the ports are removed promptly (GC remains the backstop).

**Request**

| Field | Type | Description |
|------|------|-------------|
| `agent_id` | string (UUID) | Agent resource UUID |

**Response**

| Field | Type | Description |
|------|------|-------------|
| `results` | object[] | Per-port deletion result |

Each `results[]` entry:

| Field | Type | Description |
|------|------|-------------|
| `share_id` | string | Share identifier |
| `status` | enum | `deleted`, `failed` |
| `error` | string | Present when `status=failed` |

**Behavior**

1. Lookup all shares for `agent_id` in `status in {provisioning, active, failed}`.
2. For each, run the same deletion sequence as `DeletePort`.
3. Return per-share results; any failures are retried by GC.

## PortShare Resource

Expose stores each share as a first-class resource.

| Field | Type | Description |
|------|------|-------------|
| `share_id` | string | 8-character share identifier |
| `organization_id` | string (UUID) | Owning organization (derived from agent) |
| `agent_id` | string (UUID) | Agent resource UUID |
| `thread_id` | string (UUID) | Thread where the share was created |
| `port` | int32 | Local TCP port inside the agent pod |
| `hostname` | string | `exposed-<share_id>.ziti` |
| `description` | string | Optional display text |
| `status` | enum | `provisioning`, `active`, `deleting`, `failed` |
| `last_error` | string | Error message when `status=failed` |
| `created_at` | timestamp | Creation time |
| `updated_at` | timestamp | Last update time |

## OpenZiti Provisioning Model

Expose provisions OpenZiti resources through [Ziti Management](openziti.md#ziti-management-service).

### Naming

All created resources use deterministic names derived from `share_id`:

| Resource | Name |
|----------|------|
| Service | `exposed-<share_id>` |
| Intercept hostname | `exposed-<share_id>.ziti` |
| Intercept config | `exposed-<share_id>-intercept-v1` |
| Host config | `exposed-<share_id>-host-v1` |
| Bind policy | `exposed-<share_id>-bind` |
| Dial policy | `exposed-<share_id>-dial` |

This makes provisioning and cleanup **idempotent**.

### Resources Created per Share

For each share, Expose ensures:

1. **OpenZiti service** named `exposed-<share_id>` with role attribute `exposed-services`.
2. **Intercept config** (`intercept.v1`) matching hostname `exposed-<share_id>.ziti` and TCP port `port`.
3. **Host config** (`host.v1`) forwarding to `127.0.0.1:port` inside the agent pod network namespace.
4. **Bind service policy** granting only the agent's role attribute `agent-<agent_id>` permission to bind the service.
5. **Dial service policy** granting selected user devices permission to dial the service (see [OpenZiti: Exposed Ports Access Scope](../open-questions.md#openziti-exposed-ports-access-scope)).

### Port Share Flow

```mermaid
sequenceDiagram
    participant A as Agent Pod
    participant GW as Gateway
    participant EX as Expose
    participant ZM as Ziti Management
    participant ZC as OpenZiti Controller
    participant U as User
    participant ZDE as Ziti Desktop Edge

    Note over U,ZDE: Prereq: user device enrolled in OpenZiti

    Note over A: Agent starts dev server
    A->>GW: AddPort(thread_id, port)
    GW->>EX: AddPort
    EX->>ZM: Ensure service + configs + policies
    ZM->>ZC: Create/Update Ziti resources
    ZC-->>ZM: OK
    ZM-->>EX: OK
    EX-->>A: url = http://exposed-<id>.ziti:<port>

    Note over U: User opens link
    U->>ZDE: Browser connects to exposed-<id>.ziti:<port>
    ZDE->>A: Dial Ziti service exposed-<id>
    A->>A: Sidecar forwards to 127.0.0.1:<port>
```

## Removal, Cleanup, and GC

### When port-share services are removed

A port share (and its OpenZiti service/config/policy objects) should be removed when any of the following occurs:

1. **Explicit revoke** — user (Console) or agent calls `DeletePort(share_id)`.
2. **Workload stop** — when an agent workload is terminated, the control plane should revoke any remaining shares for that agent.
   - Preferred: the **Agents Orchestrator** calls Expose to revoke shares as part of workload stop.
   - Backstop: Expose GC detects shares whose agent workload no longer exists and deletes them.
3. **Failed provisioning** — shares stuck in `failed` or `provisioning` beyond a timeout are deleted by GC.

Time-based expiry (TTL) is intentionally not specified yet; add it as a follow-up if we want automatic expiration even for long-running workloads.

### How OpenZiti resources are removed (idempotent)

Expose deletes OpenZiti resources using deterministic names. Deletion treats "not found" as success.

Recommended delete sequence for a share `exposed-<share_id>`:

1. Delete dial policy `exposed-<share_id>-dial`.
2. Delete bind policy `exposed-<share_id>-bind`.
3. Delete service `exposed-<share_id>`.
4. Delete configs `exposed-<share_id>-intercept-v1` and `exposed-<share_id>-host-v1`.

OpenZiti revocation closes active connections; deleting the service/policies makes the URL stop working.

### State machine

- On delete request, set `status=deleting`.
- Attempt OpenZiti deletions.
- On success, delete the `PortShare` record (or mark it deleted/tombstoned; retention is implementation-defined).
- On error, keep the record, set `last_error`, and rely on GC to retry.

### Garbage collection loop

Expose runs a background reconciler that:

- retries deletion for shares stuck in `failed` or `deleting`.
- deletes shares whose backing workload is no longer running (based on a control-plane workload lookup; implementation-defined).

All deletions are idempotent and may be retried safely.
