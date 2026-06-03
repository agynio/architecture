# EgressRules Service

## Overview

The EgressRules service owns the lifecycle of `EgressRule` resources and their attachments to agents. It is the control-plane counterpart to the [Egress Gateway](egress-gateway.md) — the gateway is on the data path, this service is on the configuration path. It provisions per-rule OpenZiti services via [Ziti Management](openziti.md), creates Dial policies on attachment, runs a reconciliation loop to converge actual OpenZiti state with desired state, and publishes change events through [Notifications](notifications.md) so the gateway invalidates its rule cache.

The service is structurally analogous to [Expose Service](expose-service.md) — a domain-focused service that manages a small set of OpenZiti resources per managed entity, with its own reconciliation loop. Rules and Egress Gateway are separated from the Agents service because their lifecycle (Ziti orchestration + reconciliation) is a distinct concern from agent configuration.

## Responsibilities

| Responsibility | Description |
|---|---|
| **Egress Rule CRUD** | Create, read, update, delete `EgressRule` resources. Validate matcher and effect on create/update |
| **Egress Rule Attachment CRUD** | Create, read, delete attachments binding a rule to an agent. List attachments by rule, by agent, by organization |
| **Per-rule OpenZiti service lifecycle** | On rule create, call Ziti Management to create the OpenZiti service `egress-rule-<rule_id>` with `intercept.v1` and `host.v1` configs. On rule delete, delete the service. On rule update where `matcher.domain_pattern` or `matcher.ports` changed, update the service's intercept config |
| **Per-attachment Dial policy lifecycle** | On attachment create, call Ziti Management to create a Dial policy granting `agent-<agent_id>` access to `@egress-rule-<rule_id>`. On detach, delete the policy |
| **Reconciliation** | Periodic sweep to repair drift between rule/attachment records and actual OpenZiti state |
| **Change notifications** | Publish `egress_rule.updated` and `egress_rule_attachment.updated` events to the organization's [Notifications](notifications.md) room for cache invalidation by the gateway |
| **Internal rule lookup** | Provide `ListEgressRulesByAgent(agent_id)` for the Egress Gateway data path |

## Classification

Mixed plane — control plane for CRUD (Gateway-exposed) and data plane for `ListEgressRulesByAgent` (called on the request hot path by the Egress Gateway).

| Aspect | Detail |
|---|---|
| **Plane** | Mixed (control + data) |
| **Language** | Go |
| **Repository** | `agynio/egress-rules` |
| **API** | gRPC (internal) + Gateway (external via ConnectRPC) |
| **State** | PostgreSQL — `egress_rules` and `egress_rule_attachments` tables |
| **External dependencies** | [Ziti Management](openziti.md), [Authorization](authz.md) (permission checks + agent org-membership check on attach), [Notifications](notifications.md), [Secrets](secrets.md) (existence check on `secret_id` at rule create/update) |

## API

### Egress Rule CRUD

| Method | Description |
|---|---|
| **CreateEgressRule** | Create a rule. Validates: unique `(organization_id, matcher.domain_pattern)`, no overlap with reserved zones (`*.ziti`, `*.svc`, `*.cluster.local`, `100.64.0.0/10`), header `value` xor `secret_id` per entry. Provisions the OpenZiti service `egress-rule-<rule_id>` via Ziti Management |
| **GetEgressRule** | Fetch a rule by ID |
| **ListEgressRules** | List rules in an organization. Cursor pagination |
| **UpdateEgressRule** | Update mutable fields. If `matcher.domain_pattern` or `matcher.ports` changes, updates the OpenZiti service's `intercept.v1` config |
| **DeleteEgressRule** | Delete a rule. Rejects if any attachment exists — caller must detach first. Deletes the OpenZiti service via Ziti Management |
| **ListEgressRulesByAgent** | **Internal-only.** Returns all rules attached to a given `agent_id`. Called by the Egress Gateway on cache miss |
| **CountRulesReferencingSecret** | **Internal-only.** Returns the count (and IDs) of active rules whose `effect.inject` references a given `secret_id`. Called by the [Secrets](secrets.md) service to enforce referential integrity before deleting a secret |

### Egress Rule Attachment CRUD

| Method | Description |
|---|---|
| **CreateEgressRuleAttachment** | Attach a rule to an agent. Validates: the agent belongs to the rule's organization (via an [Authorization](authz.md) check that `organization:<rule.org_id>` is the `org` of `agent:<agent_id>`), attachment is unique on `(rule_id, agent_id)`. Creates the per-attachment Dial policy via Ziti Management |
| **DeleteEgressRuleAttachment** | Detach a rule from an agent. Deletes the Dial policy |
| **ListEgressRuleAttachments** | List attachments, filterable by `rule_id` or `agent_id` |

## EgressRule Resource

See [Resource Definitions — Egress Rule](resource-definitions.md#egress-rule) for the canonical field-by-field schema.

| Field | Type | Description |
|---|---|---|
| `id` | string (UUID) | Unique identifier |
| `organization_id` | string (UUID) | Owning organization |
| `name`, `description` | string | Human-readable labels |
| `matcher` | object | `domain_pattern` (required), `ports` (default `[80, 443]`), `methods?`, `path_pattern?` |
| `effect` | object | `action?` (`allow` \| `deny` \| null), `inject?` (list of headers) |
| `openziti_service_id` | string | OpenZiti service ID for this rule |
| `created_at`, `updated_at` | timestamp | |

## EgressRuleAttachment Resource

| Field | Type | Description |
|---|---|---|
| `id` | string (UUID) | Unique identifier |
| `rule_id` | string (UUID) | Reference to the EgressRule |
| `agent_id` | string (UUID) | Reference to the Agent ([Agents service](agents-service.md)) |
| `openziti_dial_policy_id` | string | OpenZiti Dial policy ID for this attachment |
| `created_at` | timestamp | |

Unique on `(rule_id, agent_id)`. Attachments are immutable — create and delete only.

## OpenZiti Resources

For each rule, two OpenZiti resources via [Ziti Management](openziti.md):

| Resource | Ziti Management RPC | When |
|---|---|---|
| **Service** `egress-rule-<rule_id>` (with attached `intercept.v1` and `host.v1` configs, role attribute `egress-services`) | `CreateService` | On `CreateEgressRule` |
| **Service** deletion | `DeleteService` | On `DeleteEgressRule` |

For each attachment, one OpenZiti policy:

| Resource | Ziti Management RPC | When |
|---|---|---|
| **Dial policy** (`identityRoles: ["#agent-<agent_id>"]`, `serviceRoles: ["@egress-rule-<rule_id>"]`) | `CreateServicePolicy` | On `CreateEgressRuleAttachment` |
| **Dial policy** deletion | `DeleteServicePolicy` | On `DeleteEgressRuleAttachment` |

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

1. **Missing OpenZiti services for active rules.** For each `EgressRule` row, verify the corresponding OpenZiti service exists. If absent, re-create it. If present but its `intercept.v1` config drifts from the rule's `matcher`, update the config.
2. **Missing Dial policies for active attachments.** For each `EgressRuleAttachment` row, verify the corresponding Dial policy exists. If absent, re-create it.
3. **Orphaned OpenZiti services.** List OpenZiti services with role attribute `egress-services`. Any service `egress-rule-<id>` whose `<id>` does not correspond to a live `EgressRule` row → delete.
4. **Orphaned Dial policies.** List Dial policies whose `serviceRoles` reference an `egress-rule-<id>` service. Any policy whose `(agent_id, rule_id)` does not correspond to a live attachment → delete.

This ensures eventual cleanup of all OpenZiti resources regardless of transient failures or missed events.

## Notifications

Events published to the organization's [Notifications](notifications.md) room (`organization:<org_id>`):

| Event | Emitted when |
|---|---|
| `egress_rule.updated` | An `EgressRule` is created, updated, or deleted |
| `egress_rule_attachment.updated` | An `EgressRuleAttachment` is created or deleted |

The Egress Gateway subscribes per organization. On any event, it invalidates the corresponding `agent_id` rule cache(s) and refetches on next request.

## Authorization

| Operation | Check |
|---|---|
| `CreateEgressRule`, `UpdateEgressRule`, `DeleteEgressRule` | `owner` on `organization:<org_id>` |
| `GetEgressRule`, `ListEgressRules` | `member` on `organization:<org_id>` |
| `CreateEgressRuleAttachment`, `DeleteEgressRuleAttachment` | `can_edit_config` on `agent:<agent_id>` (and the rule must be in the agent's organization) |
| `ListEgressRuleAttachments` (by `agent_id`) | `can_read_config` on `agent:<agent_id>` |
| `ListEgressRuleAttachments` (by `rule_id`) | `member` on `organization:<rule.org_id>` |
| `ListEgressRulesByAgent` (internal) | Internal only — gated by Istio `AuthorizationPolicy` |
| `CountRulesReferencingSecret` (internal) | Internal only — gated by Istio `AuthorizationPolicy` |

See [Authorization — EgressRules Service](authz.md#egressrules-service) for the full reference. No new OpenFGA types are introduced; rules use the existing organization-level checks and attachments use the existing per-agent `can_edit_config`.

## Gateway Exposure

| Gateway Proto Service | Methods |
|---|---|
| `EgressRulesGateway` | `CreateEgressRule`, `GetEgressRule`, `ListEgressRules`, `UpdateEgressRule`, `DeleteEgressRule`, `CreateEgressRuleAttachment`, `DeleteEgressRuleAttachment`, `ListEgressRuleAttachments` |

`ListEgressRulesByAgent` is internal-only and not exposed through the Gateway.

## Configuration

| Field | Source | Description |
|---|---|---|
| `LISTEN_ADDRESS` | Deployment config | gRPC listen address |
| `DATABASE_URL` | Deployment config | PostgreSQL connection string |
| `ZITI_MANAGEMENT_ADDRESS` | Deployment config | gRPC address of [Ziti Management](openziti.md) |
| `AUTHORIZATION_SERVICE_ADDRESS` | Deployment config | gRPC address of [Authorization](authz.md) (permission checks + agent org-membership check on attach) |
| `SECRETS_SERVICE_ADDRESS` | Deployment config | gRPC address of [Secrets](secrets.md) (existence check on `secret_id` at rule create/update) |
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
