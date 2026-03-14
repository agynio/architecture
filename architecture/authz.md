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

All platform identities (users, agents, channels, runners) are represented as a single `identity` type in OpenFGA. Services do not need to know the identity type when constructing tuples or performing checks — they use `identity:<identity_id>` uniformly. The `identity_type` distinction (from [Authentication](authn.md)) is orthogonal to the authorization model.

### Tenant Permissions (Users)

Users have permissions within a tenant. The permission model is designed for granular extension — individual capabilities can be granted or grouped into higher-level roles as needs emerge.

| Permission | Capabilities |
|------------|-------------|
| **owner** | Full access. Manage tenant settings, membership, all resources. Delete tenant |
| **member** | Chat. View tracing. View resources (read-only) |

`owner` implies `member`. Additional granular permissions (e.g., manage agents, manage models, view tracing) can be added as relations on the `tenant` type without changing the model structure.

### Non-User Identities

Agents, channels, and runners do not have tenant-level roles. Their access is determined by resource-level relationships in the authorization graph. OpenZiti service policies restrict which services they can reach (first layer). The Authorization service enforces resource-level access (second layer).

For example, an agent can only access threads it participates in, files attached to those threads, and its own agent state. These constraints are expressed as relationships in OpenFGA, not as static role assignments.

## How Services Use Authorization

### Permission Checks

Before performing an operation, a service calls `Check` on the Authorization service:

```
Check(identity:<identity_id>, can_read, thread:<thread_id>) → allowed: bool
```

If denied, the service returns a permission error. The identity and tenant are available in gRPC metadata (see [Authentication](authn.md)).

### Relationship Writes

When state changes, the owning service writes relationship tuples through the Authorization service's `Write` method. `Write` supports atomic multi-tuple writes (adds and deletes in a single call).

## Model Deployment

The authorization model is managed as infrastructure-as-code in a dedicated repository (`agynio/openfga-model`):

| Content | Description |
|---------|-------------|
| Authorization model DSL | Type definitions, relations, computed permissions |
| Terraform module | Creates the OpenFGA store, writes the model version |
| Model tests | `fga model test` — validates expected check results against sample tuples |

The repository is versioned via git tags. The platform's infrastructure repo (`agynio/bootstrap_v2`) references a specific version:

```hcl
module "openfga_authz" {
  source  = "github.com/agynio/openfga-model?ref=v1.2.0"
  api_url = var.openfga_api_url
}
```

The [OpenFGA Terraform provider](https://registry.terraform.io/providers/openfga/openfga/latest) manages store and model lifecycle:

- `openfga_store` — creates the store, persists `store_id` in Terraform state.
- `openfga_authorization_model_document` — produces a stable JSON representation from the DSL. Output changes only on semantic changes (not formatting).
- `openfga_authorization_model` — writes a new model version. OpenFGA supports model versioning natively — each write creates a new version, old tuples continue to work.

Terraform outputs (`store_id`, `model_id`) feed into the Authorization service's configuration (environment variables via K8s config).

### Model Update Flow

1. Change the authorization model DSL in `agynio/openfga-model`.
2. Run `fga model test` to validate.
3. Merge, tag a new version.
4. Update the module version in `agynio/bootstrap_v2`.
5. `terraform apply` writes the new model version to OpenFGA.
6. Authorization service picks up the new `model_id` on next deploy (or uses latest if not pinned).

## OpenFGA Deployment

OpenFGA runs as a service within the Kubernetes cluster. It uses PostgreSQL as its data store (same infrastructure, separate database or schema).

| Aspect | Details |
|--------|---------|
| Storage | PostgreSQL |
| Protocol | gRPC |
| Deployment | Kubernetes (Helm chart) |
| Local development | Part of the bootstrap cluster |
