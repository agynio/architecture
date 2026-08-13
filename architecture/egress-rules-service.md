# EgressRules Service

## Overview

The EgressRules service owns the lifecycle of `EgressRule` resources and their attachments to agents. It is the control-plane counterpart to the [Egress Gateway](egress-gateway.md) — the gateway is on the data path, this service is on the configuration path. It provisions per-rule OpenZiti services via [Ziti Management](openziti.md), creates Dial policies on attachment, runs a reconciliation loop to converge actual OpenZiti state with desired state, and publishes change events through [Notifications](notifications.md) so the gateway invalidates its rule cache.

A rule names one of two kinds of destination, and the difference is confined to the destination end of the service's work:

| Target | `matcher` field | Service the rule's Dial policies target | Service owned by |
|---|---|---|---|
| **Public** | `domain_pattern` (+ `ports`) | `egress-rule-<rule_id>` | This service — provisioned on rule create |
| **Private** | `private_resource_id` | The resource's `private-<resource_id>` | [Networks service](networks-service.md) — referenced, never written here |

A private-target rule provisions no OpenZiti *service*; it flips the referenced resource into [gateway mediation](private-networks.md#gateway-mediation) through the Networks service instead. Attachment behaves identically for both kinds: one Dial policy per attachment, granting the target's role attribute access to the service the rule names. **For a private target that Dial policy is what grants reachability** — attaching the rule to an agent gives that agent's workloads access to the resource *and* applies the effect, in one act. Detaching removes both.

That is deliberate: an agent-principal [PrivateResourceAccess](private-networks.md#privateresourceaccess) grant and an agent attachment are authorized by the same check — `can_edit_config` on the agent plus the cross-org guard — so the unified path grants nothing the separate one did not. Grants remain the only way to reach a resource as a user, app, or group, and the only way to reach a `tcp` one; the resource's effective principal set is the union of the two. See [Private Networks — EgressRule Interaction](private-networks.md#egressrule-interaction).

Both services write Dial policies against `@private-<resource_id>` and neither disturbs the other's: reconciliation on both sides sweeps by [ownership tag](private-networks.md#openziti-resource-tagging), so an attachment policy tagged `agyn.managed_by: egress-rules-service` is invisible to the Networks service's orphan pass and vice versa.

Everything else — matcher evaluation, effect, notifications, propagation — is identical for both kinds of target.

The service is structurally analogous to [Expose Service](expose-service.md) — a domain-focused service that manages a small set of OpenZiti resources per managed entity, with its own reconciliation loop. Rules and Egress Gateway are separated from the Agents service because their lifecycle (Ziti orchestration + reconciliation) is a distinct concern from agent configuration.

## Responsibilities

| Responsibility | Description |
|---|---|
| **Egress Rule CRUD** | Create, read, update, delete `EgressRule` resources. Validate matcher, effect, and (private targets) `upstream_tls` on create/update |
| **Egress Rule Attachment CRUD** | Create, read, delete attachments binding a rule to an agent or an [environment](resource-definitions.md#environment). List attachments by rule, by agent, by environment, by organization |
| **Per-rule OpenZiti service lifecycle** | **Public targets only.** On rule create, call Ziti Management to create the OpenZiti service `egress-rule-<rule_id>` with `intercept.v1` and `host.v1` configs. On rule delete, delete the service. On rule update where `matcher.domain_pattern` or `matcher.ports` changed, update the service's intercept config |
| **Per-attachment Dial policy lifecycle** | On attachment create, call Ziti Management to create a Dial policy granting the target's identity role access to `@<openziti_service_id>` (the concrete OpenZiti service ID stored on the rule — its own service for a public target, the private resource's for a private one) — `agent-<agent_id>` for agent targets, `environment-<environment_id>` for environment targets (the [Agents Orchestrator](agents-orchestrator.md) stamps this role attribute on every workload identity it creates for a workload running the environment, agent workloads and sandboxes alike). On detach, delete the policy |
| **Private resource mediation** | On the first rule naming a [PrivateResource](private-networks.md#privateresource) and the deletion of the last one, call the [Networks service](networks-service.md)'s `SetPrivateResourceMediation` so it rebinds the resource's service to or from the Egress Gateway |
| **Reconciliation** | Periodic sweep to repair drift between rule/attachment records and actual OpenZiti state |
| **Change notifications** | Publish `egress_rule.updated` and `egress_rule_attachment.updated` events to the organization's [Notifications](notifications.md) room for cache invalidation by the gateway |
| **Internal rule lookup** | Provide `ListEgressRulesByAgent(agent_id)` and `ListEgressRulesByEnvironment(environment_id)` for the Egress Gateway data path. The gateway's effective rule set for a workload is the union of both. Private-target rules carry the referenced resource's `intercept_host` and `protocol` in the response, so the gateway needs no Networks call on the request path — the gateway invalidates those denormalized values on [`private_resource.updated`](egress-gateway.md#caching-and-invalidation) |
| **Private-resource reference lookup** | Provide `CountRulesReferencingPrivateResource` and `ListMediatedPrivateResources` for the Networks service's referential-integrity guards and mediation reconciliation |

## Classification

Mixed plane — control plane for CRUD (Gateway-exposed) and data plane for `ListEgressRulesByAgent` (called on the request hot path by the Egress Gateway).

| Aspect | Detail |
|---|---|
| **Plane** | Mixed (control + data) |
| **Language** | Go |
| **Repository** | `agynio/egress-rules` |
| **API** | gRPC (internal) + Gateway (external via ConnectRPC) |
| **State** | PostgreSQL — `egress_rules` and `egress_rule_attachments` tables |
| **External dependencies** | [Ziti Management](openziti.md), [Authorization](authz.md) (permission checks + agent org-membership check on attach), [Notifications](notifications.md), [Secrets](secrets.md) (existence check on `secret_id` at rule create/update), [Networks](networks-service.md) (private-target validation and mediation flips) |

## API

### Egress Rule CRUD

| Method | Description |
|---|---|
| **CreateEgressRule** | Create a rule. Validates: exactly one of `matcher.domain_pattern` / `matcher.private_resource_id`; header `value` xor `secret_id` per entry. Public targets additionally: no overlap with reserved zones (`*.agyn`, `*.svc`, `*.cluster.local`, `100.64.0.0/10`). Private targets additionally: the resource exists, is in the rule's organization, and has protocol `http` or `https` (via `Networks.GetPrivateResource`), and `upstream_tls` sets at most one of `ca_bundle_secret_id` / `insecure_skip_verify`. Provisions `egress-rule-<rule_id>` for a public target; calls `Networks.SetPrivateResourceMediation` for a private one. Rules are not unique per destination — several may name one domain pattern or one resource |
| **GetEgressRule** | Fetch a rule by ID |
| **ListEgressRules** | List rules in an organization. Cursor pagination, filterable by target kind and by `private_resource_id` |
| **UpdateEgressRule** | Update mutable fields. The target is immutable — a rule cannot be repointed from public to private or between resources; delete and recreate. If `matcher.domain_pattern` or `matcher.ports` changes, updates the OpenZiti service's `intercept.v1` config |
| **DeleteEgressRule** | Delete a rule. Rejects if any attachment exists — caller must detach first. Deletes the OpenZiti service via Ziti Management (public target), or calls `Networks.SetPrivateResourceMediation(mediated: false)` when it was the resource's last rule (private target) |
| **ListEgressRulesByAgent** | **Internal-only.** Returns all rules attached to a given `agent_id`. Called by the Egress Gateway on cache miss |
| **ListEgressRulesByEnvironment** | **Internal-only.** Returns all rules attached to a given `environment_id`. Called by the Egress Gateway on cache miss |
| **CountRulesReferencingSecret** | **Internal-only.** Returns the count (and IDs) of active rules whose `effect.inject` or `upstream_tls.ca_bundle_secret_id` references a given `secret_id`. Called by the [Secrets](secrets.md) service to enforce referential integrity before deleting a secret |
| **CountRulesReferencingPrivateResource** | **Internal-only.** Returns the count (and IDs) of rules whose `matcher.private_resource_id` is a given resource. Called by the [Networks service](networks-service.md) before deleting a resource or changing its `protocol` |
| **ListMediatedPrivateResources** | **Internal-only.** Returns the IDs of every private resource in an organization named by at least one rule. Called by the Networks service's reconciliation loop to re-derive desired `mediation` in one round trip per organization |
| **ListAttachedRuleDomains** | **Internal-only.** Given an `agent` or `environment` principal, return the `domain_pattern` and `ports` of every public-target rule attached to it. Called by the Networks service to detect a [hostname collision](private-networks.md#hostname-collisions) before granting that principal a resource |

### Egress Rule Attachment CRUD

| Method | Description |
|---|---|
| **CreateEgressRuleAttachment** | Attach a rule to an agent or an environment (exactly one target). Validates: the target belongs to the rule's organization (via an [Authorization](authz.md) check), attachment is unique on `(rule_id, target)`, and — for a public-target rule — that the target cannot already dial a PrivateResource whose `intercept_host` collides with the rule's `domain_pattern` (via `Networks.ListPrivateResourcesReachableBy`). See [Private Networks — Hostname collisions](private-networks.md#hostname-collisions). Creates the per-attachment Dial policy via Ziti Management |
| **DeleteEgressRuleAttachment** | Detach a rule from its target. Deletes the Dial policy |
| **ListEgressRuleAttachments** | List attachments, filterable by `rule_id`, `agent_id`, or `environment_id` |

## EgressRule Resource

See [Resource Definitions — Egress Rule](resource-definitions.md#egress-rule) for the canonical field-by-field schema.

| Field | Type | Description |
|---|---|---|
| `id` | string (UUID) | Unique identifier |
| `organization_id` | string (UUID) | Owning organization |
| `name`, `description` | string | Human-readable labels |
| `matcher` | object | `domain_pattern` xor `private_resource_id` (one required), `ports` (public only, default `[80, 443]`), `methods?`, `path_pattern?` |
| `effect` | object | `action?` (`allow` \| `deny` \| null), `inject?` (list of headers) |
| `upstream_tls` | object \| null | Private `https` targets only — `server_name?`, `ca_bundle_secret_id?` xor `insecure_skip_verify?` |
| `openziti_service_id` | string | The OpenZiti service this rule's Dial policies target: its own `egress-rule-<id>` for a public target, the referenced resource's `private-<resource_id>` for a private one. Owned by this service only in the public case |
| `created_at`, `updated_at` | timestamp | |

## EgressRuleAttachment Resource

| Field | Type | Description |
|---|---|---|
| `id` | string (UUID) | Unique identifier |
| `rule_id` | string (UUID) | Reference to the EgressRule |
| `agent_id` | string (UUID) | Target Agent ([Agents service](agents-service.md)). Mutually exclusive with `environment_id` |
| `environment_id` | string (UUID) | Target [Environment](resource-definitions.md#environment) ([Agents service](agents-service.md)). Mutually exclusive with `agent_id` |
| `openziti_dial_policy_id` | string | OpenZiti Dial policy ID for this attachment |
| `created_at` | timestamp | |

Exactly one of `agent_id` or `environment_id` is set. Unique on `(rule_id, target)`. Attachments are immutable — create and delete only.

## OpenZiti Resources

For each **public-target** rule, one OpenZiti service via [Ziti Management](openziti.md):

| Resource | Ziti Management RPC | When |
|---|---|---|
| **Service** `egress-rule-<rule_id>` (with attached `intercept.v1` and `host.v1` configs, role attribute `egress-services`) | `CreateService` | On `CreateEgressRule` |
| **Service** deletion | `DeleteService` | On `DeleteEgressRule` |

The service name is `egress-rule-<rule_id>`. The EgressRules service stores the OpenZiti service ID returned by Ziti Management as `openziti_service_id`; Dial policies target that concrete service ID with `@<openziti_service_id>`, not the service name.

A **private-target** rule creates no service. It records the referenced resource's `openziti_service_id` (read once from `Networks.GetPrivateResource` at create) so its attachment policies have a concrete ID to name, and calls `Networks.SetPrivateResourceMediation` on the first rule for that resource and on the deletion of the last.

For each attachment, one OpenZiti policy — same shape whichever kind of target the rule has:

| Resource | Ziti Management RPC | When |
|---|---|---|
| **Dial policy** (`identityRoles: ["#agent-<agent_id>"]`, `serviceRoles: ["@<openziti_service_id>"]`) | `CreateServicePolicy` | On `CreateEgressRuleAttachment` |
| **Dial policy** deletion | `DeleteServicePolicy` | On `DeleteEgressRuleAttachment` |

Every OpenZiti resource this service creates carries `agyn.managed_by: egress-rules-service` per the [tagging convention](private-networks.md#openziti-resource-tagging). This is what lets attachment policies and [PrivateResourceAccess](private-networks.md#privateresourceaccess) policies coexist on the same `private-<resource_id>` service without either reconciler treating the other's work as an orphan.

Config object shapes for `intercept.v1` and `host.v1` are in [Egress Gateway — Service Configs](egress-gateway.md#service-configs).

## Reconciliation

The EgressRules service runs a periodic reconciliation loop to repair drift between persistent state and OpenZiti reality. Mirrors the pattern from [Expose Service — Reconciliation](expose-service.md#reconciliation).

### Triggers

| Trigger | Source | Latency |
|---|---|---|
| Rule / attachment write | Synchronous in the API handler | Inline |
| Periodic reconciliation poll | Timer-based | Configurable interval (catch-all) |

### Reconciliation Logic

Each pass:

1. **Missing OpenZiti services for active public-target rules.** For each `EgressRule` row with a `domain_pattern`, verify the corresponding OpenZiti service exists. If absent, re-create it. If present but its `intercept.v1` config drifts from the rule's `matcher`, update the config.
2. **Stale service reference on private-target rules.** For each `EgressRule` row with a `private_resource_id`, verify the stored `openziti_service_id` still matches what `Networks.GetPrivateResource` reports. If the resource's service was re-created since, adopt the new ID and re-point the rule's attachment policies. If the resource is gone, mark the rule failed and surface it — the delete guard should have prevented this.
3. **Missing Dial policies for active attachments.** For each `EgressRuleAttachment` row, verify the corresponding Dial policy exists. If absent, re-create it.
4. **Orphaned OpenZiti services.** List OpenZiti services with role attribute `egress-services` tagged `agyn.managed_by: egress-rules-service`. Any service `egress-rule-<id>` whose `<id>` does not correspond to a live `EgressRule` row → delete. Mediated private-resource services carry the same role attribute but are tagged to the Networks service and are left alone.
5. **Orphaned Dial policies.** List Dial policies tagged `agyn.managed_by: egress-rules-service`. Any policy whose `(target, rule_id)` does not correspond to a live attachment → delete.

This ensures eventual cleanup of all OpenZiti resources regardless of transient failures or missed events.

## Notifications

Events published to the organization's [Notifications](notifications.md) room (`organization:<org_id>`):

| Event | Emitted when |
|---|---|
| `egress_rule.updated` | An `EgressRule` is created, updated, or deleted |
| `egress_rule_attachment.updated` | An `EgressRuleAttachment` is created or deleted |

The Egress Gateway subscribes per organization. On any event, it invalidates the corresponding `agent_id` / `environment_id` rule cache(s) and refetches on next request.

## Authorization

| Operation | Check |
|---|---|
| `CreateEgressRule`, `UpdateEgressRule`, `DeleteEgressRule` | `owner` on `organization:<org_id>`. A private target additionally requires the referenced resource to be in that organization — the same check `CreatePrivateResource` is authorized by |
| `GetEgressRule`, `ListEgressRules` | `member` on `organization:<org_id>` |
| `CreateEgressRuleAttachment`, `DeleteEgressRuleAttachment` (agent target) | `can_edit_config` on `agent:<agent_id>` (and the rule must be in the agent's organization) |
| `CreateEgressRuleAttachment`, `DeleteEgressRuleAttachment` (environment target) | `can_edit_config` on `environment:<environment_id>` — the same permission that edits the [Environment](resource-definitions.md#environment)'s other contents (and the rule must be in the environment's organization) |
| `ListEgressRuleAttachments` (by `agent_id`) | `can_read_config` on `agent:<agent_id>` |
| `ListEgressRuleAttachments` (by `environment_id`) | `member` on `organization:<org_id>` |
| `ListEgressRuleAttachments` (by `rule_id`) | `member` on `organization:<rule.org_id>` |
| `ListEgressRulesByAgent`, `ListEgressRulesByEnvironment` (internal) | Internal only — gated by Istio `AuthorizationPolicy` |
| `CountRulesReferencingSecret`, `CountRulesReferencingPrivateResource`, `ListMediatedPrivateResources` (internal) | Internal only — gated by Istio `AuthorizationPolicy` |

See [Authorization — EgressRules Service](authz.md#egressrules-service) for the full reference. This service introduces no OpenFGA types of its own: rules use organization-level checks, and each attachment uses its target's `can_edit_config` — on the [`agent`](authz.md#agent) type or the [`environment`](authz.md#environment) type.

Attaching a **private-target** rule grants the target reachability to the private resource, so it is worth stating what that costs at this layer: nothing new. `can_edit_config` on the agent or environment is precisely what [`CreatePrivateResourceAccess`](networks-service.md#authorization) already requires to grant the same principal the same access, and both paths run the same cross-org guard. The unified surface changes who *writes* the policy, not who is allowed to.

## Gateway Exposure

| Gateway Proto Service | Methods |
|---|---|
| `EgressRulesGateway` | `CreateEgressRule`, `GetEgressRule`, `ListEgressRules`, `UpdateEgressRule`, `DeleteEgressRule`, `CreateEgressRuleAttachment`, `DeleteEgressRuleAttachment`, `ListEgressRuleAttachments` |

`ListEgressRulesByAgent` and `ListEgressRulesByEnvironment` are internal-only and not exposed through the Gateway.

## Configuration

| Field | Source | Description |
|---|---|---|
| `LISTEN_ADDRESS` | Deployment config | gRPC listen address |
| `DATABASE_URL` | Deployment config | PostgreSQL connection string |
| `ZITI_MANAGEMENT_ADDRESS` | Deployment config | gRPC address of [Ziti Management](openziti.md) |
| `AUTHORIZATION_SERVICE_ADDRESS` | Deployment config | gRPC address of [Authorization](authz.md) (permission checks + agent org-membership check on attach) |
| `SECRETS_SERVICE_ADDRESS` | Deployment config | gRPC address of [Secrets](secrets.md) (existence check on `secret_id` at rule create/update) |
| `NETWORKS_SERVICE_ADDRESS` | Deployment config | gRPC address of [Networks](networks-service.md) (`GetPrivateResource` for private-target validation, `SetPrivateResourceMediation` for the mediation flip) |
| `NOTIFICATIONS_ADDRESS` | Deployment config | gRPC address of [Notifications](notifications.md) |
| `RECONCILIATION_INTERVAL` | Deployment config | How often the reconciliation loop runs (default `60s`) |

## Data Store

PostgreSQL. The EgressRules service owns its database with `egress_rules` and `egress_rule_attachments` tables.

## Implementation

| Aspect | Details |
|---|---|
| Repository | `agynio/egress-rules` |
| Language | Go |
| API framework | gRPC with ConnectRPC for the Gateway-exposed surface |
| Internal calls | Standard gRPC clients for Ziti Management, Authorization, Secrets, Notifications |
