# Authorization

## Overview

The platform uses [OpenFGA](https://openfga.dev) for fine-grained authorization. OpenFGA is a CNCF relationship-based access control (ReBAC) engine inspired by [Google Zanzibar](https://research.google/pubs/pub48190/). It evaluates access by traversing a graph of relationships between identities and resources.

A dedicated **Authorization** service sits in front of OpenFGA. All services call the Authorization service — no service communicates with OpenFGA directly.

## Authorization Service

The Authorization service is a thin gRPC proxy to OpenFGA. It centralizes the OpenFGA connection configuration (`store_id`, `model_id`, API endpoint), adds observability (metrics, tracing, structured logging), and provides a stable internal interface.

Services construct relationship tuples using their domain knowledge and send them through the Authorization service. The service does not interpret or transform tuples — it forwards them to OpenFGA with injected configuration.

### Interface

The Authorization service mirrors the OpenFGA runtime API:

| Method | Description |
|--------|-------------|
| **Check** | Can identity X perform relation Y on resource Z? Returns `allowed: bool` |
| **BatchCheck** | Multiple checks in a single call. Each check has a `correlation_id` for matching responses |
| **Write** | Write and/or delete relationship tuples atomically (up to 100 tuples per call) |
| **Read** | Read tuples matching a filter (user, relation, object — each optional). Paginated |
| **ListObjects** | What objects of type T does identity X have relation Y with? Returns list of object IDs |
| **ListUsers** | What identities have relation Y with object Z? Returns list of identity IDs |

Callers do not provide `store_id` or `authorization_model_id` — the Authorization service injects these from its own configuration.

### Classification

The Authorization service is a **data plane** service — it handles permission checks on the live request path.

## Authorization Model

The authorization model defines types, relations, and how permissions are computed. Written in OpenFGA's DSL and deployed via Terraform (see [Model Deployment](#model-deployment)).

### Identities

All platform identities (users, agents, runners, apps) are represented as a single `identity` type in OpenFGA. Services do not need to know the identity type when constructing tuples or performing checks — they use `identity:<identity_id>` uniformly. The `identity_type` distinction (from [Authentication](authn.md)) is orthogonal to the authorization model.

Any identity can hold any relationship that is modeled in OpenFGA. See [Identity](identity.md) for the identity registry.

### Organization Permissions

Identities have permissions within an organization via OpenFGA relationship tuples. The permission model is designed for granular extension — individual capabilities can be granted or grouped into higher-level roles as needs emerge.

| Permission | Capabilities |
|------------|-------------|
| **owner** | Full access. Manage organization settings, membership, all resources. Delete organization |
| **member** | Chat. View tracing. View resources (read-only) |

`owner` implies `member`. Additional granular permissions (e.g., manage agents, manage models, view tracing) can be added as relations on the `organization` type without changing the model structure.

Any identity type can hold organization permissions. For example, an agent that creates an organization becomes its owner.

## How Services Use Authorization

### Permission Checks

Before performing an operation, a service calls `Check` on the Authorization service:

```
Check(identity:<identity_id>, can_read, thread:<thread_id>) → allowed: bool
```

If denied, the service returns a permission error. The identity is available in gRPC metadata (see [Authentication](authn.md)). Organization context, when needed, is passed as a request parameter.

### Relationship Writes

When state changes, the owning service writes relationship tuples through the Authorization service's `Write` method. `Write` supports atomic multi-tuple writes (adds and deletes in a single call).

## Model Deployment

The authorization model is managed as infrastructure-as-code in the Authorization service repo (`agynio/authorization`), under `terraform/`:

| Content | Description |
|---------|-------------|
| Authorization model DSL | Type definitions, relations, computed permissions |
| Terraform module | Creates the OpenFGA store, writes the model version |
| Model tests | `fga model test` — validates expected check results against sample tuples |

Terraform is applied directly from `agynio/authorization/terraform/` to manage the OpenFGA store and model versions.

The [OpenFGA Terraform provider](https://registry.terraform.io/providers/openfga/openfga/latest) manages store and model lifecycle:

- `openfga_store` — creates the store, persists `store_id` in Terraform state.
- `openfga_authorization_model_document` — produces a stable JSON representation from the DSL. Output changes only on semantic changes (not formatting).
- `openfga_authorization_model` — writes a new model version. OpenFGA supports model versioning natively — each write creates a new version, old tuples continue to work.

Terraform outputs (`store_id`, `model_id`) feed into the Authorization service's configuration (environment variables via K8s config).

### Model Update Flow

1. Change the authorization model DSL in `agynio/authorization/terraform/`.
2. Run `fga model test` to validate.
3. Merge.
4. `terraform apply` from `agynio/authorization/terraform/` writes the new model version to OpenFGA.
5. Authorization service picks up the new `model_id` on next deploy (or uses latest if not pinned).

## OpenFGA Deployment

OpenFGA runs as a service within the Kubernetes cluster. It uses PostgreSQL as its data store (same infrastructure, separate database or schema).

| Aspect | Details |
|--------|---------|
| Storage | PostgreSQL |
| Protocol | gRPC |
| Deployment | Kubernetes (Helm chart) |
| Local development | Part of the bootstrap cluster |

## Cluster Permissions

In addition to organization-scoped permissions, the platform has cluster-level permissions for administrative operations that span all organizations.

### Cluster Type

The authorization model includes a `cluster` type with a singleton object `cluster:global`:

| Permission | Capabilities |
|------------|-------------|
| **admin** | Register and manage cluster-scoped [apps](apps.md). Register and manage cluster-scoped runners. Platform administration |

### Tuples

Cluster admin permissions are stored as relationship tuples in OpenFGA:

```
identity:<userId>, admin, cluster:global
```

### Bootstrap

The initial cluster admin is seeded during platform bootstrap. Terraform writes directly to PostgreSQL:

1. Terraform creates a user record in the Users service database. This is a platform-only user — not associated with any OIDC identity.
2. Terraform registers the user's identity in the Identity service database.
3. Terraform creates an API token for this user (writes to `user_api_tokens` table — hash of the generated token).
4. Terraform writes the OpenFGA tuple: `identity:<userId>, admin, cluster:global`.

The generated API token is stored as a Terraform output (sensitive) and is used for cluster-level operations — registering cluster-scoped apps and runners.

### Usage

Services that handle cluster-level operations check the `admin` relation on `cluster:global`:

```
Check(identity:<callerId>, admin, cluster:global) → allowed: bool
```

If denied, the service returns a permission error.
