# LLM Service

## Overview

The LLM service manages LLM providers, models, and subscriptions as internal resources, and provides resolution for the [LLM Proxy](llm-proxy.md). It is the single source of truth for LLM credentials, protocol selection, and model-to-provider mappings.

Agents do not interact with the LLM service directly. They call the [LLM Proxy](llm-proxy.md), which resolves through this service.

The service answers two resolution questions, one per [LLM mode](resource-definitions.md#environment):

| Mode | Question | Method |
|---|---|---|
| `platform` | "which provider and remote name does this platform model ID mean?" | [`ResolveModel`](#resolvemodel) |
| `native` | "which vendor credential does this workload use?" | [`ResolveSubscription`](#resolvesubscription) |

## Responsibilities

| Responsibility | Description |
|---------------|-------------|
| **Provider CRUD** | Create, read, update, delete LLM provider resources |
| **Model CRUD** | Create, read, update, delete model resources |
| **Subscription CRUD** | Create, read, update, delete [Subscription](providers.md#subscription) resources |
| **Subscription attachment CRUD** | Create, delete, and list [attachments](providers.md#subscription-attachment) binding a subscription to an agent or an environment |
| **Model resolution** | Resolve a model ID to provider endpoint, token, remote model name, and organization ID — consumed by the [LLM Proxy](llm-proxy.md) |
| **Subscription resolution** | Resolve a calling workload and vendor to a token, upstream, and injected headers — consumed by the [LLM Proxy](llm-proxy.md) |

## Classification

The LLM service is a **data plane** service — model resolution is on the hot path during agent execution.

## Model Resolution

The `ResolveModel` method returns everything the [LLM Proxy](llm-proxy.md) needs to forward a request to an external LLM provider.

### ResolveModel

**Request:**

| Field | Type | Description |
|-------|------|-------------|
| `model_id` | string (UUID) | Platform model ID |

**Response:**

| Field | Type | Description |
|-------|------|-------------|
| `endpoint` | string | Provider base URL (e.g., `https://api.openai.com`) |
| `token` | string | Provider authentication token. Set when `auth_method` is `bearer` or `x_api_key`; empty when `custom_headers` |
| `headers` | map<string, string> | Custom request headers. Set when `auth_method` is `custom_headers`; empty otherwise |
| `remote_name` | string | Model identifier on the provider's side (e.g., `gpt-5`, `claude-sonnet-4-20250514`) |
| `protocol` | string | LLM API protocol — `responses` or `anthropic_messages`. From the provider's [`protocol`](providers.md#llm-provider) field |
| `auth_method` | string | Authentication method — `bearer`, `x_api_key`, or `custom_headers`. From the provider's [`authMethod`](providers.md#llm-provider) field |
| `organization_id` | string (UUID) | Organization that owns the model |

### Resolution Chain

```
Model.id → Model.llmProvider → LLM Provider (endpoint + token) + Model.remoteName
```

The LLM service looks up the model, follows the provider reference, and returns the combined result.

## Subscription Resolution

`ResolveSubscription` answers the `native`-mode question: given the workload that is calling and the vendor whose host it addressed, which credential does it use? Nothing in an intercepted request identifies a credential — the request is the agent CLI's own, unmodified — so the key is the caller, not the payload.

### ResolveSubscription

**Request:**

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | string (UUID) | The agent class the calling workload runs. Empty for a [sandbox](resource-definitions.md#sandbox), which has none |
| `environment_id` | string (UUID) | The environment the calling workload runs |
| `vendor` | string | `claude` or `codex` — determined by the host the caller addressed |

Both identifiers come from the caller's OpenZiti identity via [`ResolveIdentity`](openziti.md#identity-resolution); neither is self-asserted by the workload.

**Response:**

| Field | Type | Description |
|-------|------|-------------|
| `subscription_id` | string (UUID) | The resolved subscription. The proxy has no other source for it, and [metering](llm-proxy.md#native-mode-labels) records it as `resource_id` |
| `token` | string | The resolved credential, fetched from the [Secret](providers.md#secret) the subscription references |
| `account_id` | string | Vendor account identifier, when the vendor requires one; empty otherwise |
| `upstream_endpoint` | string | Where the proxy forwards — fixed per vendor |
| `protocol` | string | `responses` or `anthropic_messages` — fixed per vendor |
| `allowed_models` | list<string> | The environment's [`llm_allowed_models`](resource-definitions.md#environment), read by this service from the [Agents](agents-service.md) service. Empty means no restriction |
| `organization_id` | string (UUID) | Organization that owns the subscription |

**Everything the proxy needs to serve a native-mode connection is on this response.** That is deliberate: it is what keeps the proxy a forwarder with no configuration lookups of its own, and it is what makes [`environment_id` on the identity](openziti.md#identity-resolution) sufficient — the proxy passes the id through and never reads an environment.

The consequence is that this service, not the proxy, resolves the environment. It already fans out to [Secrets](secrets.md) on every resolution; adding [Agents](agents-service.md) alongside costs one more call on a path that runs once per connection rather than once per request. Putting the same call in the proxy would put it on the request path of the LLM hot path and contradict the property above.

`llm_allowed_models` lives on the environment rather than on the subscription because one subscription attaches to many environments, and the point of that is that they may differ. A restriction stored on the credential would apply everywhere it is used.

Note what is *not* here: the vendor's [`placeholder_env`](providers.md#vendors). The proxy strips `Authorization` and `x-api-key` unconditionally, whatever they were called on the way in, so it never needs the name. Its only consumer is the [Agents Orchestrator](agents-orchestrator.md#workload-spec-assembly), which reads it from the [attachment listing](#subscription-management) at workload assembly.

Returns `NOT_FOUND` when no subscription for that vendor is attached at either scope. The LLM Proxy turns this into a platform error the caller can read, rather than forwarding an unauthenticated request.

### Resolution Chain

```
(agent_id, environment_id, vendor)
  → SubscriptionAttachment (agent scope, else environment scope)
  → Subscription (vendor → upstream + protocol + header shape)
  → Secret → token
  → Environment → allowed_models
```

The agent scope is consulted first and shadows the environment's for the same vendor. Because attachments are unique on `(vendor, target)`, at most one candidate exists at each scope and the chain has no ambiguity to resolve.

The token is fetched through the [Secrets](secrets.md) service on each resolution. That call is not on the per-request path — the proxy resolves a binding once per connection.

## Authorization

LLM providers, models, and subscriptions are org-scoped. Both resolution methods are internal operations called by the LLM Proxy via Istio — they have no OpenFGA check.

| Operation | Check |
|-----------|-------|
| Create, Update, Delete (providers, models, subscriptions) | `owner` on `organization:<org_id>` |
| Get, List (providers, models, subscriptions) | `member` on `organization:<org_id>` |
| Attach, Detach (subscriptions) | `owner` on `organization:<org_id>`, plus `can_edit_config` on the target agent or environment |
| `ResolveModel` | Internal only — LLM Proxy via Istio |
| `ResolveSubscription` | Internal only — LLM Proxy via Istio |

A subscription resolves for a workload because someone attached it, and attaching required edit rights on the target. There is no second check at call time: in `native` mode the attachment **is** the authorization. This is the one asymmetry with `platform` mode, where the proxy additionally checks `can_use` on the model — there is no model resource in `native` mode to check anything against.

Restricting *which* models a native-mode workload may ask for is a guardrail evaluated in the [LLM Proxy](llm-proxy.md#native-mode), which sees the model name in the request body, not an access-control decision made here.

### Referential Integrity with Secrets

`DeleteSecret` calls `CountSubscriptionsReferencingSecret(secret_id)` on this service before deleting, and rejects the delete when any subscription references it — the same treatment the [EgressRules service](egress-rules-service.md) already gets, and not a dependency cycle for the same reason: Secrets calls out only on delete, the LLM service calls in only on subscription create/update (existence check via `ResolveSecretExists`).

### Change Notifications

The LLM service publishes `subscription.updated` and `subscription_attachment.updated` events to the flat `llm_subscriptions` [Notifications](notifications.md) room, so the LLM Proxy can drop cached bindings. Without it, a detached or rotated subscription stays live on already-established connections for as long as the agent CLI keeps them open — hours, in a sandbox. This is the same invalidation path the [Egress Gateway](egress-gateway.md) uses for its rule cache, and the same room shape: `egress_rules`, not `egress_rules:{org_id}`.

The room is flat rather than per-organization because a notification subscription's rooms are fixed when the stream opens. The proxy serves whichever organizations connect to it and cannot enumerate them in advance, so a per-organization room would leave it either re-subscribing on every new organization or missing invalidations for one it has not seen. Bindings live on connections rather than in a keyed cache, so there is nothing to evict selectively anyway — every open connection re-resolves on its next request.

Those two events do not cover the whole binding. `allowed_models` comes from the environment, which changes under [`environment.updated`](agents-service.md#notifications) — an event the [Agents](agents-service.md) service already publishes on the `environment:{id}` room, and now publishes to a flat `environments` room alongside it for the same reason. **The proxy subscribes to it as well**, and tightening an allowlist therefore takes effect on open connections rather than at the next one. A restriction that waits for a long-lived sandbox connection to close is not a restriction; the event already exists, so there is no reason to accept the weaker semantics.

On `CreateModel`, the LLM Service writes the tuple `organization:<org_id>, org, model:<model_id>` to the Authorization service. On `DeleteModel`, it deletes the same tuple. This grants org members implicit `can_use` on the model via the computed relation defined on the `model` type.

See [Authorization — LLM Service](authz.md#llm-service) for the full reference.

## Provider Management

CRUD operations for LLM provider resources. See [Providers, Models, and Secrets](providers.md#llm-provider) for the resource definition.

## Model Management

CRUD operations for model resources. See [Providers, Models, and Secrets](providers.md#model) for the resource definition.

## Subscription Management

CRUD operations for subscriptions and their attachments. See [Providers, Models, and Secrets — Subscription](providers.md#subscription) for the resource definitions.

| Method | Description |
|---|---|
| `CreateSubscription` | Create a subscription. Validates the vendor against the closed set and the `secret_id` via `Secrets.ResolveSecretExists`. Returns `UNIMPLEMENTED` for a vendor whose binding is [not yet closed](providers.md#vendors) — better a clear refusal than a record that can never produce a working workload |
| `GetSubscription` / `ListSubscriptions` | Read. The referenced secret is reported as a reference, never a resolved value |
| `UpdateSubscription` | Change `name`, `secret_id`, or `account_id`. `vendor` is immutable — changing it would silently redirect every workload the subscription is attached to |
| `DeleteSubscription` | Refused while any attachment exists; the error names them |
| `CreateSubscriptionAttachment` | Attach to an agent or an environment (exactly one). Rejects a target in another organization, and rejects a second subscription for the same vendor on that target |
| `DeleteSubscriptionAttachment` | Detach |
| `ListSubscriptionAttachments` | Filterable by `subscription_id`, `agent_id`, or `environment_id`. Each entry carries the subscription's `vendor` and its `placeholder_env`, which is everything the [Agents Orchestrator](agents-orchestrator.md#workload-spec-assembly) needs at workload assembly — role attributes and the placeholder — without a second call or a vendor table of its own |
| `CountSubscriptionsReferencingSecret` | **Internal only.** Called by the [Secrets](secrets.md) service before deleting a secret |

Unlike [egress rule attachments](egress-rules-service.md#openziti-resources), a subscription attachment provisions no OpenZiti resources. Native-mode interception is static infrastructure gated by role attributes the [Agents Orchestrator](agents-orchestrator.md#workload-spec-assembly) stamps at workload identity creation — so this service owns credential lifecycle only, with no reconciliation loop and no Ziti Management dependency.
