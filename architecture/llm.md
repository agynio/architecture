# LLM Service

## Overview

The LLM service manages LLM providers and models as internal resources, and provides model resolution for the [LLM Proxy](llm-proxy.md). It is the single source of truth for provider credentials, protocol selection, and model-to-provider mappings.

Agents do not interact with the LLM service directly. They call the [LLM Proxy](llm-proxy.md), which resolves models through this service.

## Responsibilities

| Responsibility | Description |
|---------------|-------------|
| **Provider CRUD** | Create, read, update, delete LLM provider resources |
| **Model CRUD** | Create, read, update, delete model resources |
| **Model resolution** | Resolve a model ID to provider endpoint, token, remote model name, and organization ID — consumed by the [LLM Proxy](llm-proxy.md) |

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
| `token` | string | Provider authentication token |
| `remote_name` | string | Model identifier on the provider's side (e.g., `gpt-5`, `claude-sonnet-4-20250514`) |
| `protocol` | string | LLM API protocol — `responses` or `anthropic_messages`. From the provider's [`protocol`](providers.md#llm-provider) field |
| `auth_method` | string | Authentication method — `bearer` or `x_api_key`. From the provider's [`authMethod`](providers.md#llm-provider) field |
| `organization_id` | string (UUID) | Organization that owns the model |

### Resolution Chain

```
Model.id → Model.llmProvider → LLM Provider (endpoint + token) + Model.remoteName
```

The LLM service looks up the model, follows the provider reference, and returns the combined result.

## Authorization

LLM providers and models are org-scoped. `ResolveModel` is an internal operation called by the LLM Proxy via Istio — it has no OpenFGA check.

| Operation | Check |
|-----------|-------|
| Create, Update, Delete (providers, models) | `owner` on `organization:<org_id>` |
| Get, List (providers, models) | `member` on `organization:<org_id>` |
| `ResolveModel` | Internal only — LLM Proxy via Istio |

On `CreateModel`, the LLM Service writes the tuple `organization:<org_id>, org, model:<model_id>` to the Authorization service. On `DeleteModel`, it deletes the same tuple. This grants org members implicit `can_use` on the model via the computed relation defined on the `model` type.

See [Authorization — LLM Service](authz.md#llm-service) for the full reference.

## Provider Management

CRUD operations for LLM provider resources. See [Providers, Models, and Secrets](providers.md#llm-provider) for the resource definition.

## Model Management

CRUD operations for model resources. See [Providers, Models, and Secrets](providers.md#model) for the resource definition.
