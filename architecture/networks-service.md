# Networks Service

## Overview

The Networks service owns the lifecycle of `Network`, `TunnelCredential`, `PrivateResource`, and `PrivateResourceAccess` resources. It is the control-plane companion to the OpenZiti-overlay routing described in [Private Networks](private-networks.md). It provisions per-network and per-resource OpenZiti resources via [Ziti Management](openziti.md), creates Dial policies on access grants, polls Controller session info for tunnel liveness, runs a reconciliation loop to converge actual OpenZiti state with desired state, and publishes change events through [Notifications](notifications.md).

Structurally analogous to the [EgressRules service](egress-rules-service.md) — a domain-focused service that manages a small set of OpenZiti resources per managed entity, with its own reconciliation loop.

## Responsibilities

| Responsibility | Description |
|---|---|
| **Network CRUD** | Create, read, update, delete `Network` resources. Provisions the per-network Bind policy on create; deletes the policy on delete |
| **TunnelCredential CRUD** | Issue and revoke tunnel enrollment credentials. Each credential maps 1:1 to an OpenZiti identity with role attributes `["tunnels", "network-<id>"]` |
| **PrivateResource CRUD** | Create, read, update, delete `PrivateResource` resources. Validates intercept hostname against reserved zones and the per-org uniqueness constraint on `(intercept_host, intercept_port)`. Provisions the OpenZiti service + configs |
| **PrivateResourceAccess CRUD** | Create and delete access grants. Each grant materializes as an OpenZiti Dial policy. List grants by resource, by principal, by organization |
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
| **External dependencies** | [Ziti Management](openziti.md), [Authorization](authz.md) (permission checks + agent / group org-membership checks on grant), [Groups](groups-service.md) (existence checks on `group_id` principals), [Notifications](notifications.md) |

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
| **CreateTunnelCredential** | Issue a new tunnel credential. Creates an OpenZiti identity with role attributes `["tunnels", "network-<id>"]` and returns the enrollment JWT (one-time, not persisted in plaintext) |
| **ListTunnelCredentials** | List credentials in a network, with current `enrolled_at` and `last_seen_at` |
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

The Networks service tracks per-credential liveness by polling the OpenZiti Controller's per-identity session API:

1. Periodically (configurable interval, default 30s), list active sessions for all identities with the `tunnels` role attribute.
2. For each `TunnelCredential` whose `openziti_identity_id` has an active session, update `last_seen_at` and (if not yet set) `enrolled_at`.
3. On transition between online and offline, emit `tunnel_status.changed` to the organization's notifications room.

The Controller's session info is authoritative — the Networks service does not maintain a separate heartbeat protocol with the tunneler. This avoids duplicating OpenZiti's existing session tracking and works with any standard OpenZiti tunneler distribution (Linux binary, Docker, k8s helm chart, desktop apps).

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
4. **Orphaned OpenZiti services.** List services with any `network-resources-<id>` role attribute. Any `private-<id>` whose ID does not correspond to a live `PrivateResource` row → delete.
5. **Orphaned Dial policies.** List Dial policies whose `serviceRoles` reference a `private-<id>` service. Any policy whose `(resource_id, principal_type, principal_id)` does not correspond to a live grant → delete.
6. **Orphaned Tunnel identities.** List identities with the `tunnels` role attribute. Any identity whose ID does not correspond to a live `TunnelCredential` row → delete.
7. **Orphaned Bind policies.** List Bind policies whose `identityRoles` reference a `network-<id>` role attribute. Any policy whose network ID does not correspond to a live `Network` row → delete.

This ensures eventual cleanup of all OpenZiti resources regardless of transient failures or missed events.

## Notifications

Events published to the organization's [Notifications](notifications.md) room (`organization:<org_id>`):

| Event | Emitted when |
|---|---|
| `network.updated` | A `Network` is created, updated, or deleted |
| `tunnel_credential.updated` | A `TunnelCredential` is created or deleted |
| `tunnel_status.changed` | A Tunnel transitions online/offline (sourced from Controller session info) |
| `private_resource.updated` | A `PrivateResource` is created, updated, or deleted |
| `private_resource_access.updated` | A `PrivateResourceAccess` is created or deleted |

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
| `CreatePrivateResourceAccess` (user or group principal) | `owner` on `organization:<org_id>` + resource org-membership check |
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
| `NOTIFICATIONS_ADDRESS` | Deployment config | gRPC address of [Notifications](notifications.md) |
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
- [Groups Service](groups-service.md) — Group lifecycle and OpenZiti role-attribute sync (group principals on access grants)
- [OpenZiti Integration](openziti.md) — overlay infrastructure and Ziti Management RPCs
- [EgressRules Service](egress-rules-service.md) — sibling service with the same structural pattern
