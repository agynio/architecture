# Networks Service

## Overview

The Networks service owns the lifecycle of `Network`, `TunnelCredential`, `PrivateResource`, and `PrivateResourceAccess` resources. It is the control-plane companion to the OpenZiti-overlay routing described in [Private Networks](private-networks.md). It provisions per-network and per-resource OpenZiti resources via [Ziti Management](openziti.md), creates Dial policies on access grants, polls Controller session info for tunnel liveness, runs a reconciliation loop to converge actual OpenZiti state with desired state, and publishes change events through [Notifications](notifications.md).

Structurally analogous to the [EgressRules service](egress-rules-service.md) — a domain-focused service that manages a small set of OpenZiti resources per managed entity, with its own reconciliation loop.

## Responsibilities

| Responsibility | Description |
|---|---|
| **Network CRUD** | Create, read, update, delete `Network` resources. Provisions the per-network Bind policy on create; deletes the policy on delete |
| **TunnelCredential CRUD** | Issue and revoke tunnel enrollment credentials. Each credential maps 1:1 to an OpenZiti identity with role attributes `["tunnels", "network-<id>"]` |
| **PrivateResource CRUD** | Create, read, update, delete `PrivateResource` resources. Validates intercept hostname against reserved zones (`*.agyn`, `*.svc`, `*.cluster.local`, `100.64.0.0/10`, `localhost`, `127.0.0.0/8`, `::1/128`) and the per-org uniqueness constraint per port in `intercept_ports`. Provisions the OpenZiti service + configs with the [tagging convention](private-networks.md#openziti-resource-tagging) |
| **Gateway mediation** | Materialize a resource's `mediation` state on request from the [EgressRules service](egress-rules-service.md): rebind `private-<id>` between the network's Tunnels and the [Egress Gateway](egress-gateway.md), maintain the `private-<id>-upstream-<intercept_port>` services and the gateway's Dial policies on them. See [Private Networks — Gateway Mediation](private-networks.md#gateway-mediation) |
| **PrivateResourceAccess CRUD** | Create and delete access grants. Each grant materializes as exactly one OpenZiti Dial policy. Principals: `agent`, `environment`, `user`, `app`, `group`. List grants by resource, by principal, by organization |
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
| **External dependencies** | [Ziti Management](openziti.md), [Authorization](authz.md) (caller-permission checks + cross-org guard via OpenFGA `org` relation on the principal — covers `agent`, `user`, `app`, and `group` principals uniformly), [Identity](identity.md) (existence check + type validation for individual principals `agent`/`user`/`app` via `GetIdentityType`), [Groups](groups-service.md) (existence check for `group` principals via `GetGroup`), [Agents](agents-service.md) (existence and organization check for `environment` principals via `GetEnvironment` — an environment is a configuration resource, not an identity, so it is not in the Identity registry), [EgressRules](egress-rules-service.md) (hostname-collision check on access grants, referential-integrity guards on delete and protocol change, mediation reconciliation), [Messaging](messaging.md) (event-bus publication + subscription), [Notifications](notifications.md) (client-facing UI updates) |

The [EgressRules](egress-rules-service.md) dependency runs both ways — that service calls `GetPrivateResource`, `SetPrivateResourceMediation`, and `ListPrivateResourcesReachableBy` here. Both directions are request-scoped and neither is on a startup path, so the mutual dependency does not sequence deployment. It exists because the two services own opposite halves of one destination: this one owns the resource's OpenZiti objects, that one owns the policy that decides whether the gateway sits in front of them.

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
| **GetPrivateResource** | Fetch a resource by ID. Also called internally by the [EgressRules service](egress-rules-service.md) to validate a rule's private target |
| **ListPrivateResources** | List resources, filterable by `network_id` or `organization_id`. Cursor pagination |
| **UpdatePrivateResource** | Update mutable fields. If `target_host`/`target_ports` or `intercept_host`/`intercept_ports` change, updates the corresponding OpenZiti config. Changing `protocol` to `tcp` is rejected while any rule names the resource (`EgressRules.CountRulesReferencingPrivateResource`) |
| **DeletePrivateResource** | Delete a resource. Rejected while any rule names it, listing the rules — same referential-integrity shape the [Secrets](secrets.md) service uses. Otherwise cascades to all access grants, deletes the OpenZiti service and attached configs, and (if mediated) the upstream services and the gateway's Dial policies on them |
| **SetPrivateResourceMediation** | **Internal-only.** Set a resource's `mediation` to `tunnel` or `egress_gateway` and materialize the OpenZiti change. Called by the [EgressRules service](egress-rules-service.md) when a resource gains its first rule or loses its last. Idempotent — the desired state is the argument, not a delta |
| **ListPrivateResourcesReachableBy** | **Internal-only.** Given an `agent` or `environment` principal, return the resources it can dial by grant, with their `intercept_host` and `intercept_ports`. Called by the [EgressRules service](egress-rules-service.md) to detect a [hostname collision](private-networks.md#hostname-collisions) before attaching a public-target rule |

### PrivateResourceAccess CRUD

| Method | Description |
|---|---|
| **CreatePrivateResourceAccess** | Grant access. Validates: principal exists (for `agent` / `user` / `app` via `Identity.GetIdentityType` — also confirms `principal_type` matches the registered identity type; for `group` via `Groups.GetGroup`; for `environment` via `Agents.GetEnvironment`); the principal belongs to the resource's organization (cross-org `Check` via [Authorization](authz.md) `org` relation on `<principal_type>:<principal_id>`, which the [`environment` type](authz.md#environment) carries like every other principal); the grant is unique on `(resource_id, principal_type, principal_id)`; and — for `agent` and `environment` principals — that no public egress rule attached to that principal claims a `domain_pattern` colliding with the resource's `intercept_host` (via `EgressRules.ListAttachedRuleDomains`). Creates the per-grant Dial policy |
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

For each **mediated** PrivateResource, one upstream service per intercept port with a gateway Dial policy on each, plus a rewrite of the front service. Created on `SetPrivateResourceMediation(egress_gateway)`, reversed on `SetPrivateResourceMediation(tunnel)`:

| Resource | Ziti Management RPC | Effect |
|---|---|---|
| **Front service** `private-<resource_id>` | `UpdateService` | Role attribute `network-resources-<network_id>` → `egress-services`; `host.v1` replaced with the forwarding shape |
| **Service** `private-<resource_id>-upstream-<intercept_port>`, one per entry in `intercept_ports` (static `host.v1` naming `target_host` and the paired `target_ports` entry, role attribute `network-resources-<network_id>`) | `CreateService` / `DeleteService` | The tunnel-bound legs the gateway dials. The network's existing Bind policy covers them via the role attribute, and the per-service target port is what applies the intercept→target mapping |
| **Dial policy** per upstream service (`identityRoles: ["#egress-gateway-hosts"]`, `serviceRoles: ["@<id>"]`) | `CreateServicePolicy` / `DeleteServicePolicy` | Lets the Egress Gateway reach the tunnel |

`UpdatePrivateResource` on a mediated resource that changes `intercept_ports` or `target_ports` recreates the upstream services to match the new pairing, since each encodes one mapping.

Config shapes for both mediation states are in [Private Networks — OpenZiti resources per mediation state](private-networks.md#openziti-resources-per-mediation-state).

For each PrivateResourceAccess, one OpenZiti Dial policy:

| Resource | Ziti Management RPC | When |
|---|---|---|
| **Dial policy** (`identityRoles: ["#<principal-role>"]`, `serviceRoles: ["@private-<resource_id>"]`) | `CreateServicePolicy` | On `CreatePrivateResourceAccess` |
| **Dial policy** deletion | `DeleteServicePolicy` | On `DeletePrivateResourceAccess` |

The principal role attribute is one of `agent-<id>`, `environment-<id>`, `user-<id>`, `app-<id>`, or `group-<id>` per [Private Networks — Dial Policy](private-networks.md#dial-policy-per-access-grant). None of them is provisioned here: every one is stamped on the identity by the service that owns it, and a grant only writes the policy that matches it.

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
3. **Drifted mediation.** Per organization, call `EgressRules.ListMediatedPrivateResources` and compare against the `mediation` column. Apply the flip where they differ; where they agree, verify the state's objects (front-service role attribute and `host.v1`, upstream service, gateway Dial policy) and repair drift. One round trip per organization, not per resource.
4. **Missing Dial policies for live grants.** For each `PrivateResourceAccess` row, verify the corresponding Dial policy exists. If absent, re-create it.
5. **Orphaned OpenZiti services.** List services tagged `agyn.managed_by: networks-service, agyn.resource_type: private_resource | private_resource_upstream`. Any whose `agyn.resource_id` does not correspond to a live `PrivateResource` row → delete; any upstream service whose resource is no longer mediated → delete.
6. **Orphaned Dial policies.** List policies tagged `agyn.managed_by: networks-service, agyn.resource_type: resource_access | gateway_dial_policy`. Any whose `agyn.resource_id` does not correspond to a live `PrivateResourceAccess` row, or to a still-mediated resource, → delete. Attachment policies on the same services are tagged to the [EgressRules service](egress-rules-service.md) and are outside this filter.
7. **Orphaned Tunnel identities.** List identities tagged `agyn.managed_by: networks-service, agyn.resource_type: tunnel_credential`. Any whose `agyn.resource_id` does not correspond to a live `TunnelCredential` row → delete.
8. **Orphaned Bind policies.** List policies tagged `agyn.managed_by: networks-service, agyn.resource_type: network_bind_policy`. Any whose `agyn.network_id` does not correspond to a live `Network` row → delete.

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

**There is no equivalent cleanup for a deleted environment**, because the [Agents](agents-service.md) service publishes no environment lifecycle event to subscribe to. The grant is left behind, and it is inert: `environment-<id>` is stamped only on workloads running that environment, and a deleted environment has none and can get none — [deletion is blocked](resource-definitions.md#environment) while any agent or sandbox references it. A Dial policy naming a role attribute no identity carries grants nothing. This is a hygiene gap, not an access one, and closing it wants an `agyn.agents.environment.deleted` event rather than a per-pass existence check on every grant.

## Client-Facing Updates

Separately from the service-to-service event bus, the Networks service publishes UI-facing updates to the organization's [Notifications](notifications.md) room (`organization:<org_id>`) for Console reactivity:

| Event | Emitted when |
|---|---|
| `network.updated` | A `Network` is created, updated, or deleted |
| `tunnel_credential.updated` | A `TunnelCredential` is created or deleted |
| `tunnel_status.changed` | A Tunnel transitions online/offline |
| `private_resource.updated` | A `PrivateResource` is created, updated, or deleted — including a [mediation](private-networks.md#gateway-mediation) flip |
| `private_resource_access.updated` | A `PrivateResourceAccess` is created or deleted |

These are fire-and-forget [Notifications](notifications.md) updates for the browser, distinct from the durable event bus. See [Messaging — Overview](messaging.md#overview) for the distinction.

One has a second consumer: the [Egress Gateway](egress-gateway.md#caching-and-invalidation) subscribes to `private_resource.updated` to invalidate the resource fields it caches alongside private-target rules, the same way it already consumes `egress_rule.updated`. Delivery is best-effort, which is why nothing on the routing path depends on those cached values.

## Authorization

| Operation | Check |
|---|---|
| `CreateNetwork`, `UpdateNetwork`, `DeleteNetwork` | `owner` on `organization:<org_id>` |
| `GetNetwork`, `ListNetworks` | `member` on `organization:<org_id>` |
| `CreateTunnelCredential`, `DeleteTunnelCredential` | `owner` on `organization:<org_id>` |
| `GetTunnelCredential`, `ListTunnelCredentials` | `member` on `organization:<org_id>` |
| `CreatePrivateResource`, `UpdatePrivateResource`, `DeletePrivateResource` | `owner` on `organization:<org_id>` |
| `GetPrivateResource`, `ListPrivateResources` | `member` on `organization:<org_id>` |
| `CreatePrivateResourceAccess` (agent principal) | `can_edit_config` on `agent:<agent_id>` + resource org-membership check |
| `CreatePrivateResourceAccess` (environment principal) | `can_edit_config` on `environment:<environment_id>` + resource org-membership check — the same permission that edits the environment's other contents, and the same one an [egress rule attachment](egress-rules-service.md#authorization) to an environment requires |
| `CreatePrivateResourceAccess` (user, app, or group principal) | `owner` on `organization:<org_id>` + resource org-membership check |
| `DeletePrivateResourceAccess` | Same check as the corresponding `CreatePrivateResourceAccess` |
| `ListPrivateResourceAccess` | `member` on `organization:<org_id>` |
| `SetPrivateResourceMediation` (internal) | Internal only — gated by Istio `AuthorizationPolicy` restricted to the [EgressRules service](egress-rules-service.md)'s ServiceAccount. The caller-facing check happened on the rule that triggered it |

See [Authorization — Networks Service](authz.md#networks-service) for the full reference. No new OpenFGA types are introduced; the resource layer uses the existing organization-level checks and per-agent `can_edit_config`.

## Gateway Exposure

| Gateway Proto Service | Methods |
|---|---|
| `NetworksGateway` | `CreateNetwork`, `GetNetwork`, `ListNetworks`, `UpdateNetwork`, `DeleteNetwork`, `CreateTunnelCredential`, `GetTunnelCredential`, `ListTunnelCredentials`, `DeleteTunnelCredential`, `CreatePrivateResource`, `GetPrivateResource`, `ListPrivateResources`, `UpdatePrivateResource`, `DeletePrivateResource`, `CreatePrivateResourceAccess`, `DeletePrivateResourceAccess`, `ListPrivateResourceAccess` |

`SetPrivateResourceMediation` is internal-only and not exposed through the Gateway — mediation is derived from the rules that exist, never set by a caller.

## Configuration

| Field | Source | Description |
|---|---|---|
| `LISTEN_ADDRESS` | Deployment config | gRPC listen address |
| `DATABASE_URL` | Deployment config | PostgreSQL connection string |
| `ZITI_MANAGEMENT_ADDRESS` | Deployment config | gRPC address of [Ziti Management](openziti.md) |
| `AUTHORIZATION_SERVICE_ADDRESS` | Deployment config | gRPC address of [Authorization](authz.md) |
| `GROUPS_SERVICE_ADDRESS` | Deployment config | gRPC address of [Groups](groups-service.md) (existence checks for `group_id` principals on access grants) |
| `AGENTS_SERVICE_ADDRESS` | Deployment config | gRPC address of [Agents](agents-service.md) (existence checks for `environment` principals on access grants) |
| `EGRESS_RULES_SERVICE_ADDRESS` | Deployment config | gRPC address of [EgressRules](egress-rules-service.md) (collision and referential-integrity checks, mediation reconciliation) |
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
