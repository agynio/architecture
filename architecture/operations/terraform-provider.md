# Terraform Provider

The **Agyn Terraform provider** is the recommended way to configure agents and their dependencies as code.

**Repository:** [`agynio/terraform-provider-agyn`](https://github.com/agynio/terraform-provider-agyn)

The provider connects to the [Gateway](../gateway.md) using a generated gRPC client from the gateway proto definitions. ConnectRPC's gRPC protocol support means the provider communicates with the same Gateway endpoint used by all other clients — no separate API surface.

## Resources

| Terraform Resource | API Resource | Description |
|-------------------|-------------|-------------|
| `agyn_user` | User | Platform user (OIDC subject, profile, cluster role). See [Users — Admin User Management](../users.md#admin-user-management) |
| `agyn_app` | App | App registration (slug, name, icon). Returns service token for enrollment |
| `agyn_runner` | Runner | Runner registration (name, organization scope, labels). Returns service token for enrollment |
| `agyn_agent` | Agent | Agent definition (identity, model, image, compute resources, configuration) |
| `agyn_volume` | Volume | Volume definition (persistent/ephemeral, mount path, size) |
| `agyn_volume_attachment` | Volume Attachment | Relationship between a volume and a container (agent, MCP, or hook) |
| `agyn_image_pull_secret` | Image Pull Secret | Registry credential (registry hostname, username, password/token). Managed by the Secrets service |
| `agyn_image_pull_secret_attachment` | Image Pull Secret Attachment | Relationship between an image pull secret and a container (agent, MCP, or hook) |
| `agyn_llm_provider` | LLM Provider | Connection to an external LLM service (endpoint, protocol, auth method, credentials). Managed by the LLM service |
| `agyn_llm_model` | Model | Platform model mapped to a remote model on an LLM provider. Managed by the LLM service |
| `agyn_mcp` | MCP | MCP server definition (image, command, compute resources) |
| `agyn_skill` | Skill | Skill definition (name, body) |
| `agyn_hook` | Hook | Hook definition (event, function, image, compute resources) |
| `agyn_env` | ENV | Environment variable (name, plain value or secret reference) |
| `agyn_init_script` | InitScript | Initialization script (shell script content) |
| `agyn_membership` | Membership | Organization membership (identity, organization, role). See [Organizations — Members Management](../organizations.md#members-management) |

## Data Sources

| Terraform Data Source | API Method | Description |
|----------------------|------------|-------------|
| `data.agyn_user` | `GetUserByOIDCSubject` or `GetUser` | Look up an existing user by OIDC subject or identity ID. Returns `identity_id` and profile. Used to reference users that were provisioned outside Terraform (e.g., via OIDC login) |

## User Resource

The `agyn_user` resource creates and manages platform users via [Users — Admin User Management](../users.md#admin-user-management). Requires cluster admin authentication.

### Schema

| Field | Type | Required | Computed | Mutable | Description |
|-------|------|----------|----------|---------|-------------|
| `identity_id` | string | — | yes | — | Platform identity UUID. Primary identifier |
| `oidc_subject` | string | yes | — | no (forces replacement) | OIDC subject claim (`sub`). Must be unique. Immutable — changing it destroys and recreates the user |
| `username` | string | optional | yes (when omitted) | yes | Cluster-wide handle. If omitted, derived from `oidc_subject` per [Users — Derivation at Provisioning](../users.md#derivation-at-provisioning). Must match `^[a-z0-9_-]+$`, max 32 chars, unique cluster-wide |
| `name` | string | optional | — | yes | Display name |
| `photo_url` | string | optional | — | yes | Profile photo URL |
| `cluster_role` | string | optional | — | yes | `admin` or unset. Controls `cluster:global admin` authorization tuple |

### Lifecycle

| Operation | API Method | Notes |
|-----------|------------|-------|
| **Create** | `CreateUser` | Creates user record, registers identity, optionally writes cluster admin tuple. Fails with `AlreadyExists` if `oidc_subject` or `username` is already taken |
| **Read** | `GetUser` | Reads by `identity_id` |
| **Update** | `UpdateUser` | Updates `username`, `name`, `photo_url`, `cluster_role` |
| **Delete** | `DeleteUser` | Deletes user record, identity registration, cluster admin tuple (if any), and all organization memberships |
| **Import** | — | Import by `identity_id` |

### Example

```hcl
resource "agyn_user" "alice" {
  oidc_subject = "auth0|abc123"
  name         = "Alice"
  cluster_role = "admin"
}
```

## User Data Source

The `data.agyn_user` data source looks up an existing user by OIDC subject or identity ID. Requires cluster admin authentication. Returns an error if the user does not exist.

Exactly one of `oidc_subject` or `identity_id` must be provided.

### Schema

| Field | Type | Direction | Description |
|-------|------|-----------|-------------|
| `oidc_subject` | string | input (Optional) | OIDC subject claim to look up. Calls `GetUserByOIDCSubject` |
| `identity_id` | string | input/output (Optional input, always Computed) | Platform identity UUID. As input: calls `GetUser`. Always present in output |
| `username` | string | output (Computed) | Cluster-wide handle |
| `name` | string | output (Computed) | Display name |
| `photo_url` | string | output (Computed) | Profile photo URL |
| `cluster_role` | string | output (Computed) | Cluster role (`admin` or empty) |

### Example

```hcl
data "agyn_user" "alice" {
  oidc_subject = "auth0|abc123"
}

resource "agyn_organization" "eng" {
  name = "Engineering"
}

resource "agyn_membership" "alice_eng" {
  organization_id = agyn_organization.eng.id
  identity_id     = data.agyn_user.alice.identity_id
  role            = "owner"
}
```

## Provisioning Resources

Some resources produce a **service token** on creation. The token is returned only once (on `terraform apply`) and stored in Terraform state. It is used to enroll the service at startup. See [Authentication — Service Tokens](../authn.md#service-tokens).

| Resource | Token Output | Enrollment |
|----------|-------------|------------|
| `agyn_app` | `service_token` | App presents token at startup → receives OpenZiti identity |
| `agyn_runner` | `service_token` | Runner presents token at startup → receives OpenZiti identity |

## LLM Provider Resource

The `agyn_llm_provider` resource creates and manages [LLM Provider](../providers.md#llm-provider) records via the `LLMGateway`. Requires org owner or cluster admin authentication.

### Schema

| Field | Type | Required | Computed | Mutable | Description |
|-------|------|----------|----------|---------|-------------|
| `id` | string | — | yes | — | Provider UUID. Primary identifier |
| `organization_id` | string | yes | — | no (forces replacement) | Organization that owns the provider |
| `name` | string | yes | — | yes | Human-readable provider name |
| `endpoint` | string | yes | — | yes | Provider base URL (e.g., `https://api.openai.com`, `https://api.anthropic.com`) |
| `protocol` | string | yes | — | yes | `responses` or `anthropic_messages` |
| `auth_method` | string | yes | — | yes | `bearer`, `x_api_key`, or `custom_headers` |
| `token` | string (sensitive) | conditional | — | yes | Required when `auth_method` is `bearer` or `x_api_key`. Must be absent when `custom_headers` |
| `headers` | map<string, string> (sensitive) | conditional | — | yes | Required (non-empty) when `auth_method` is `custom_headers`. Must be absent for `bearer` and `x_api_key`. Reserved header names (`Host`, `Content-Length`, `Connection`, `Transfer-Encoding`) are rejected at plan-time validation |

The provider validates the `token` / `headers` invariants in `ValidateConfig` so misconfigurations fail at `terraform plan`, not on apply.

### Lifecycle

| Operation | API Method | Notes |
|-----------|------------|-------|
| **Create** | `CreateProvider` | Returns the new provider's `id` |
| **Read** | `GetProvider` | Reads by `id`. `token` and `headers` are returned masked from the API and reconciled from Terraform state — drift on these fields is intentionally not detected |
| **Update** | `UpdateProvider` | Updates `name`, `endpoint`, `protocol`, `auth_method`, `token`, `headers` |
| **Delete** | `DeleteProvider` | Fails if any `agyn_llm_model` still references the provider |
| **Import** | — | Import by `<organization_id>/<id>`. `token` / `headers` must be supplied manually after import |

### Examples

```hcl
# OpenAI-compatible provider
resource "agyn_llm_provider" "openai" {
  organization_id = agyn_organization.eng.id
  name            = "OpenAI"
  endpoint        = "https://api.openai.com"
  protocol        = "responses"
  auth_method     = "bearer"
  token           = var.openai_api_key
}

# Anthropic provider (x-api-key)
resource "agyn_llm_provider" "anthropic" {
  organization_id = agyn_organization.eng.id
  name            = "Anthropic"
  endpoint        = "https://api.anthropic.com"
  protocol        = "anthropic_messages"
  auth_method     = "x_api_key"
  token           = var.anthropic_api_key
}

# Internal gateway requiring multiple custom headers
resource "agyn_llm_provider" "internal_gateway" {
  organization_id = agyn_organization.eng.id
  name            = "Internal LLM Gateway"
  endpoint        = "https://llm-gw.internal.example.com"
  protocol        = "responses"
  auth_method     = "custom_headers"
  headers = {
    "Authorization" = "Token ${var.gateway_token}"
    "x-org-id"      = agyn_organization.eng.id
  }
}
```

## Model Resource

The `agyn_llm_model` resource creates and manages [Model](../providers.md#model) records via the `LLMGateway`.

### Schema

| Field | Type | Required | Computed | Mutable | Description |
|-------|------|----------|----------|---------|-------------|
| `id` | string | — | yes | — | Model UUID |
| `organization_id` | string | yes | — | no (forces replacement) | Organization that owns the model |
| `name` | string | yes | — | yes | Internal display name (e.g., `claude-sonnet`) |
| `llm_provider_id` | string | yes | — | yes | Reference to an `agyn_llm_provider` |
| `remote_name` | string | yes | — | yes | Model identifier on the provider's side (e.g., `claude-sonnet-4-20250514`) |

### Example

```hcl
resource "agyn_llm_model" "claude_sonnet" {
  organization_id = agyn_organization.eng.id
  name            = "claude-sonnet"
  llm_provider_id = agyn_llm_provider.anthropic.id
  remote_name     = "claude-sonnet-4-20250514"
}
```

## Agent Resource Structure

Agent resources (everything except `agyn_app`, `agyn_runner`, `agyn_user`, and `agyn_membership`) share a common envelope:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID, computed |
| `description` | string | Human-readable description, optional |

Resource-specific fields are exposed as typed HCL attributes. The canonical schema for each resource is documented in [Resource Definitions](../resource-definitions.md).

### Ownership

Most sub-resources (MCP, Skill, Hook, ENV, InitScript) have an ownership field (`agent_id`, `mcp_id`, or `hook_id`) that determines which parent resource they belong to. These are required, immutable after creation, and expressed as standard Terraform resource attributes — not as attachment resources.
