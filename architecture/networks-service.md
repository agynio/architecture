# Networks Service

## Overview

The Networks service owns the lifecycle of `Network`, `TunnelCredential`, `PrivateResource`, and `PrivateResourceAccess` resources. It is the control-plane companion to the OpenZiti-overlay routing described in [Private Networks](private-networks.md). It provisions per-network and per-resource OpenZiti resources via [Ziti Management](openziti.md), creates Dial policies on access grants, polls Controller session info for tunnel liveness, runs a reconciliation loop to converge actual OpenZiti state with desired state, and publishes change events through [Notifications](notifications.md).

Structurally analogous to the [EgressRules service](egress-rules-service.md) — a domain-focused service that manages a small set of OpenZiti resources per managed entity, with its own reconciliation loop.

## Responsibilities

| Responsibility | Description |
|---|---|
| **Network CRUD** | Create, read, update, delete `Network` resources. Provisions the per-network Bind policy on create; deletes the policy on delete |
| **TunnelCredential CRUD** | Issue and revoke tunnel enrollment credentials. Each credential maps 1:1 to an OpenZiti identity with role attributes `["tunnels", "network-<id>"]` |
| **PrivateResource CRUD** | Create, read, update, delete `PrivateResource` resources. Validates intercept hostname against reserved zones (`*.ziti`, `*.svc`, `*.cluster.local`, `100.64.0.0/10`, `localhost`, `127.0.0.0/8`, `::1/128`) and the per-org uniqueness constraint per port in `intercept_ports`. Provisions the OpenZiti service + configs with the [tagging convention](private-networks.md#openziti-resource-tagging) |
| **PrivateResourceAccess CRUD** | Create and delete access grants. Each grant materializes as exactly one OpenZiti Dial policy. Principals: `agent`, `user`, `app`, `group`. List grants by resource, by principal, by organization |
| **Tunnel liveness tracking** | Poll the OpenZiti Controller for per-identity session info; surface `enrolled_at` and `last_seen_at` on `TunnelCredential`; emit `tunnel_status.changed` notifications on transitions |
| **Reconciliation** | Periodic sweep to repair drift between resource records and actual OpenZiti state |
| **Change notifications** | Publish `network.updated`, `tunnel_credential.updated`, `tunnel_status.changed`, `private_resource.updated`, `private_resource_access.updated` events to the organization's [Notifications](notifications.md) room |

## Classification

Control plane — Gateway-exposed CRUD with periodic reconciliation. Not on a request hot path.

| Aspect | Detail |
|---|---|
| **Plane** | Control |
| **Language** | Go |
| **Repository** | `agynio/networks` |
| **API** | gRPC (internal) + Gateway (external via ConnectRPC) |
| **State** | PostgreSQL — `networks`, `tunnel_credentials`, `private_resources`, `private_resource_access` tables |
| **External dependencies** | [Ziti Management](openziti.md), [Authorization](authz.md) (permission checks + agent / group org-membership checks on grant), [Groups](groups-service.md) (existence checks on `group_id` principals), [Messaging](messaging.md) (event-bus publication + subscription), [Notifications](notifications.md) (client-facing UI updates) |

## API

### Network CRUD

| Method | Description |
|---|---|
| **CreateNetwork** | Create a network. Provisions the per-network Bind policy via Ziti Management |
| **GetNetwork** | Fetch a network by ID |
| **ListNetworks** | List networks in an organization. Cursor pagination |
| **UpdateNetwork** | Update mutable fields (`name`, `description`) |
| **DeleteNetwork** | Delete a network. Cascades through all TunnelCredentials, PrivateResources, and PrivateResourceAccess grants in the network (see [Deletion Semantics](private-networks.md#deletion-semantics)) |

### TunnelCredential CRUD

| Method | Description |
|---|---|
| **CreateTunnelCredential** | Issue a new tunnel credential. Creates an OpenZiti identity with role attributes `["tunnels", "network-<id>"]`, returns the `enrollment_jwt` in the response (one-time), and sets `enrollment_jwt_revealed = true`. The JWT is not persisted in plaintext after issuance |
| **GetTunnelCredential** | Fetch a credential by ID. **Omits the `enrollment_jwt` field** — only available in the `CreateTunnelCredential` response |
| **ListTunnelCredentials** | List credentials in a network, with current `enrollment_state`, `connectivity`, `enrolled_at`, and `last_seen_at`. **Omits `enrollment_jwt`** |
| **DeleteTunnelCredential** | Delete a tunnel credential. Deletes the corresponding OpenZiti identity. Any tunneler holding the credential loses its session immediately |

### PrivateResource CRUD

| Method | Description |
|---|---|
| **CreatePrivateResource** | Create a resource. Validates: `intercept_host` not in reserved zones, unique `(org_id, intercept_host, intercept_port)`, `target_ports` cardinality matches `intercept_ports`. Provisions the OpenZiti service `private-<id>` with attached `host.v1` and `intercept.v1` configs |
| **GetPrivateResource** | Fetch a resource by ID |
| **ListPrivateResources** | List resources, filterable by `network_id` or `organization_id`. Cursor pagination |
| **UpdatePrivateResource** | Update mutable fields. If `target_host`/`target_ports` or `intercept_host`/`intercept_ports` change, updates the corresponding OpenZiti config |
| **DeletePrivateResource** | Delete a resource. Cascades to all access grants. Deletes the OpenZiti service and attached configs |

### PrivateResourceAccess CRUD

| Method | Description |
|---|---|
| **CreatePrivateResourceAccess** | Grant access. Validates: the resource belongs to the principal's organization (cross-org `Check` via [Authorization](authz.md)), the principal exists, the grant is unique on `(resource_id, principal_type, principal_id)`. Creates the per-grant Dial policy |
| **DeletePrivateResourceAccess** | Revoke access. Deletes the Dial policy |
| **ListPrivateResourceAccess** | List grants, filterable by `resource_id`, by `principal_type` + `principal_id`, or by `network_id` |

## Resource Schemas

See [Private Networks — Resource Shapes](private-networks.md#resource-shapes) for the summary tables and [Resource Definitions](resource-definitions.md) for the canonical field-by-field schemas.

## OpenZiti Resources

For each Network, one OpenZiti policy via [Ziti Management](openziti.md):

| Resource | Ziti Management RPC | When |
|---|---|---|
| **Bind policy** `network-<network_id>-bind` (`identityRoles: ["#network-<network_id>"]`, `serviceRoles: ["@network-resources-<network_id>"]`) | `CreateServicePolicy` | On `CreateNetwork` |
| **Bind policy** deletion | `DeleteServicePolicy` | On `DeleteNetwork` |

For each TunnelCredential, one OpenZiti identity:

| Resource | Ziti Management RPC | When |
|---|---|---|
| **Tunnel identity** (`type: Device`, `roleAttributes: ["tunnels", "network-<network_id>"]`, `enrollment.ott`) | `CreateTunnelIdentity` | On `CreateTunnelCredential` |
| **Tunnel identity** deletion | `DeleteTunnelIdentity` | On `DeleteTunnelCredential` |

For each PrivateResource, one OpenZiti service with attached configs:

| Resource | Ziti Management RPC | When |
|---|---|---|
| **Service** `private-<resource_id>` (with `host.v1` and `intercept.v1` configs, role attribute `network-resources-<network_id>`) | `CreateService` | On `CreatePrivateResource` |
| **Service config update** | `UpdateService` (config edits) | On `UpdatePrivateResource` when target or intercept changes |
| **Service** deletion | `DeleteService` | On `DeletePrivateResource` |

For each PrivateResourceAccess, one OpenZiti Dial policy:

| Resource | Ziti Management RPC | When |
|---|---|---|
| **Dial policy** (`identityRoles: ["#<principal-role>"]`, `serviceRoles: ["@private-<resource_id>"]`) | `CreateServicePolicy` | On `CreatePrivateResourceAccess` |
| **Dial policy** deletion | `DeleteServicePolicy` | On `DeletePrivateResourceAccess` |

The principal role attribute is one of `agent-<id>`, `user-<id>`, or `group-<id>` per [Private Networks — Dial Policy](private-networks.md#dial-policy-per-access-grant).

Service config shapes (`host.v1`, `intercept.v1`) are in [Private Networks — Per-Resource OpenZiti Service](private-networks.md#per-resource-openziti-service).

## Tunnel Liveness

The Networks service tracks per-credential liveness by polling the OpenZiti Controller's Edge Management identity API. The Controller is the authoritative source — the Networks service does not maintain a separate heartbeat protocol with the tunneler. This works with any standard OpenZiti tunneler distribution (Linux binary, Docker, k8s helm chart, desktop apps).

### Source fields

`GET /edge/management/v1/identities/<id>` returns the per-identity state used to derive `TunnelCredential` liveness fields:

| Local field | Source | Derivation |
|---|---|---|
| `enrollment_state` | `enrollment.state` on the identity | `"enrolled"` once the identity completes JWT enrollment; `"pending"` before. Set to `enrolled` on the first poll that observes the change |
| `enrolled_at` | derived | Set to `now()` the first time the Controller reports `enrollment.state == "enrolled"` |
| `connectivity` | `hasEdgeRouterConnection` field | `"online"` when `true`; `"offline"` when `false` |
| `last_seen_at` | derived | Set to `now()` on each poll that observes `hasEdgeRouterConnection: true` |

### Polling

Every `TUNNEL_LIVENESS_INTERVAL` (default 30s), the Networks service:

1. Lists identities with the `tunnels` role attribute via `GET /edge/management/v1/identities?filter=roleAttributes contains "tunnels"`.
2. For each returned identity, updates the corresponding `TunnelCredential` row's `enrollment_state`, `connectivity`, `last_seen_at`, and (on first enrollment) `enrolled_at`.
3. Detects transitions: a credential whose `connectivity` flipped this pass emits `agyn.networks.tunnel.online` or `agyn.networks.tunnel.offline` on the event bus, and the corresponding `tunnel_status.changed` to the org's [Notifications](notifications.md) room.

The 30-second poll gives ≤30s detection latency for tunnel state changes. Tighter detection would require either webhook from the Controller (not standardized in OpenZiti) or a higher poll rate (acceptable but unnecessary for v1).

## Reconciliation

The Networks service runs a periodic reconciliation loop to repair drift between persistent state and OpenZiti reality. Mirrors the pattern from [EgressRules service — Reconciliation](egress-rules-service.md#reconciliation).

### Triggers

| Trigger | Source | Latency |
|---|---|---|
| Network / TunnelCredential / PrivateResource / PrivateResourceAccess write | Synchronous in the API handler | Inline |
| Periodic reconciliation poll | Timer-based | Configurable interval (default 60s) |

### Reconciliation Logic

Each pass:

1. **Missing per-network Bind policies.** For each `Network` row, verify the corresponding `network-<id>-bind` policy exists. If absent, re-create it.
2. **Missing OpenZiti services for live resources.** For each `PrivateResource` row, verify the corresponding `private-<id>` service exists with the correct role attribute. If absent, re-create it. If its `intercept.v1` or `host.v1` config drifts from the resource record, update the config.
3. **Missing Dial policies for live grants.** For each `PrivateResourceAccess` row, verify the corresponding Dial policy exists. If absent, re-create it.
4. **Orphaned OpenZiti services.** List services tagged `agyn.managed_by: networks-service, agyn.resource_type: private_resource`. Any whose `agyn.resource_id` does not correspond to a live `PrivateResource` row → delete.
5. **Orphaned Dial policies.** List policies tagged `agyn.managed_by: networks-service, agyn.resource_type: resource_access`. Any whose `agyn.resource_id` does not correspond to a live `PrivateResourceAccess` row → delete.
6. **Orphaned Tunnel identities.** List identities tagged `agyn.managed_by: networks-service, agyn.resource_type: tunnel_credential`. Any whose `agyn.resource_id` does not correspond to a live `TunnelCredential` row → delete.
7. **Orphaned Bind policies.** List policies tagged `agyn.managed_by: networks-service, agyn.resource_type: network_bind_policy`. Any whose `agyn.network_id` does not correspond to a live `Network` row → delete.

All reconciliation walks OpenZiti resources by tag, not by name conventions — see [Private Networks — OpenZiti Resource Tagging](private-networks.md#openziti-resource-tagging). Walks are bounded by the network's resource counts, not by total platform resource count.

This ensures eventual cleanup of all OpenZiti resources regardless of transient failures or missed events.

## Events Published

Durable service-to-service events on the platform [event bus](messaging.md). Stream: `AGYN_NETWORKS`.

| Subject | Schema | Published when |
|---|---|---|
| `agyn.networks.tunnel.online` | `agyn.networks.v1.TunnelOnlineEvent` | A `TunnelCredential`'s identity transitions to having an active OpenZiti session (per Controller poll) |
| `agyn.networks.tunnel.offline` | `agyn.networks.v1.TunnelOfflineEvent` | A `TunnelCredential`'s identity transitions to having no active OpenZiti session |
| `agyn.networks.access.granted` | `agyn.networks.v1.PrivateResourceAccessGrantedEvent` | A `PrivateResourceAccess` row is created and its Dial policy is provisioned |
| `agyn.networks.access.revoked` | `agyn.networks.v1.PrivateResourceAccessRevokedEvent` | A `PrivateResourceAccess` row is deleted and its Dial policy is removed |

Known consumers (informational):

- [Tracing](tracing.md) (future) — correlate tunnel availability with private-resource span outcomes
- Audit log service (future) — append-only audit of grant lifecycle

## Events Consumed

| Subject filter | Durable consumer | Purpose |
|---|---|---|
| `agyn.groups.group.deleted` | `networks-group-cleanup` | When a [Group](groups-service.md) is deleted, delete every `PrivateResourceAccess` row with `principal_type=group, principal_id=<deleted>` and remove the backing OpenZiti Dial policies |

The handler re-reads the affected grants from the local database (the event payload carries only `group_id`), deletes them transactionally, and removes the OpenZiti Dial policies via Ziti Management. Idempotent — re-running on duplicate delivery has no additional effect (grants already deleted).

## Client-Facing Updates

Separately from the service-to-service event bus, the Networks service publishes UI-facing updates to the organization's [Notifications](notifications.md) room (`organization:<org_id>`) for Console reactivity:

| Event | Emitted when |
|---|---|
| `network.updated` | A `Network` is created, updated, or deleted |
| `tunnel_credential.updated` | A `TunnelCredential` is created or deleted |
| `tunnel_status.changed` | A Tunnel transitions online/offline |
| `private_resource.updated` | A `PrivateResource` is created, updated, or deleted |
| `private_resource_access.updated` | A `PrivateResourceAccess` is created or deleted |

These are fire-and-forget Socket.IO-style updates for the browser. They are distinct from the durable event bus and not consumed by other services. See [Messaging — Overview](messaging.md#overview) for the distinction.

## Authorization

| Operation | Check |
|---|---|
| `CreateNetwork`, `UpdateNetwork`, `DeleteNetwork` | `owner` on `organization:<org_id>` |
| `GetNetwork`, `ListNetworks` | `member` on `organization:<org_id>` |
| `CreateTunnelCredential`, `DeleteTunnelCredential` | `owner` on `organization:<org_id>` |
| `ListTunnelCredentials` | `member` on `organization:<org_id>` |
| `CreatePrivateResource`, `UpdatePrivateResource`, `DeletePrivateResource` | `owner` on `organization:<org_id>` |
| `GetPrivateResource`, `ListPrivateResources` | `member` on `organization:<org_id>` |
| `CreatePrivateResourceAccess` (agent principal) | `can_edit_config` on `agent:<agent_id>` + resource org-membership check |
| `CreatePrivateResourceAccess` (user, app, or group principal) | `owner` on `organization:<org_id>` + resource org-membership check |
| `DeletePrivateResourceAccess` | Same check as the corresponding `CreatePrivateResourceAccess` |
| `ListPrivateResourceAccess` | `member` on `organization:<org_id>` |

See [Authorization — Networks Service](authz.md#networks-service) for the full reference. No new OpenFGA types are introduced; the resource layer uses the existing organization-level checks and per-agent `can_edit_config`.

## Gateway Exposure

| Gateway Proto Service | Methods |
|---|---|
| `NetworksGateway` | `CreateNetwork`, `GetNetwork`, `ListNetworks`, `UpdateNetwork`, `DeleteNetwork`, `CreateTunnelCredential`, `ListTunnelCredentials`, `DeleteTunnelCredential`, `CreatePrivateResource`, `GetPrivateResource`, `ListPrivateResources`, `UpdatePrivateResource`, `DeletePrivateResource`, `CreatePrivateResourceAccess`, `DeletePrivateResourceAccess`, `ListPrivateResourceAccess` |

All RPCs are external; the Networks service has no internal-only methods today.

## Configuration

| Field | Source | Description |
|---|---|---|
| `LISTEN_ADDRESS` | Deployment config | gRPC listen address |
| `DATABASE_URL` | Deployment config | PostgreSQL connection string |
| `ZITI_MANAGEMENT_ADDRESS` | Deployment config | gRPC address of [Ziti Management](openziti.md) |
| `AUTHORIZATION_SERVICE_ADDRESS` | Deployment config | gRPC address of [Authorization](authz.md) |
| `GROUPS_SERVICE_ADDRESS` | Deployment config | gRPC address of [Groups](groups-service.md) (existence checks for `group_id` principals on access grants) |
| `NATS_URL` | Deployment config | NATS connection URL for the platform [event bus](messaging.md) |
| `NOTIFICATIONS_ADDRESS` | Deployment config | gRPC address of [Notifications](notifications.md) (client-facing UI updates) |
| `RECONCILIATION_INTERVAL` | Deployment config | How often the reconciliation loop runs (default `60s`) |
| `TUNNEL_LIVENESS_INTERVAL` | Deployment config | How often the Controller is polled for tunnel session info (default `30s`) |

## Data Store

PostgreSQL. The Networks service owns its database with `networks`, `tunnel_credentials`, `private_resources`, and `private_resource_access` tables.

## Implementation

| Aspect | Details |
|---|---|
| Repository | `agynio/networks` |
| Language | Go |
| API framework | gRPC with ConnectRPC for the Gateway-exposed surface |
| Internal calls | Standard gRPC clients for Ziti Management, Authorization, Groups, Notifications |

## Related Architecture

- [Private Networks](private-networks.md) — feature overview, resource model, OpenZiti topology
- [Groups Service](groups-service.md) — Group lifecycle and the `agyn.groups.group.deleted` event this service consumes
- [Messaging](messaging.md) — platform event bus contract for service-to-service async events
- [OpenZiti Integration](openziti.md) — overlay infrastructure and Ziti Management RPCs
- [EgressRules Service](egress-rules-service.md) — sibling service with the same structural pattern
