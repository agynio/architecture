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
| `agyn_secret_provider` | Secret Provider | Secret provider connection (type, provider-specific config). See [Providers, Models, and Secrets — Secret Provider](../providers.md#secret-provider) |
| `agyn_secret` | Secret | Secret value (local encrypted or remote provider reference). See [Providers, Models, and Secrets — Secret](../providers.md#secret) |
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
| `name` | string | optional | — | yes | Display name |
| `nickname` | string | optional | — | yes | Short name or handle |
| `photo_url` | string | optional | — | yes | Profile photo URL |
| `cluster_role` | string | optional | — | yes | `admin` or unset. Controls `cluster:global admin` authorization tuple |

### Lifecycle

| Operation | API Method | Notes |
|-----------|------------|-------|
| **Create** | `CreateUser` | Creates user record, registers identity, optionally writes cluster admin tuple. Fails with `AlreadyExists` if `oidc_subject` is already taken |
| **Read** | `GetUser` | Reads by `identity_id` |
| **Update** | `UpdateUser` | Updates `name`, `nickname`, `photo_url`, `cluster_role` |
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
| `name` | string | output (Computed) | Display name |
| `nickname` | string | output (Computed) | Short name or handle |
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

## Secret Provider Resource

The `agyn_secret_provider` resource creates and manages secret provider connections via [SecretsGateway](../gateway.md). Requires org owner or cluster admin authorization.

### Schema

| Field | Type | Required | Computed | Mutable | Description |
|-------|------|----------|----------|---------|-------------|
| `id` | string | — | yes | — | UUID identifier |
| `organization_id` | string | yes | — | no (forces replacement) | Organization scope |
| `type` | string | yes | — | no (forces replacement) | Provider type. Supported: `vault` |
| `address` | string | yes | — | yes | Provider server address (e.g., `http://vault:8200`) |
| `token` | string | yes | — | yes | Authentication token. Sensitive |

### Lifecycle

| Operation | API Method | Notes |
|-----------|------------|-------|
| **Create** | `CreateSecretProvider` | Creates the provider connection |
| **Read** | `GetSecretProvider` | Reads by `id` |
| **Update** | `UpdateSecretProvider` | Updates `address`, `token` |
| **Delete** | `DeleteSecretProvider` | Deletes the provider. Fails if secrets reference it |
| **Import** | — | Import by `id` |

### Example

```hcl
resource "agyn_secret_provider" "vault" {
  organization_id = agyn_organization.eng.id
  type            = "vault"
  address         = "http://vault:8200"
  token           = var.vault_token
}
```

## Secret Resource

The `agyn_secret` resource creates and manages secrets via [SecretsGateway](../gateway.md). Requires org owner or cluster admin authorization. Secrets store sensitive values either locally (encrypted at rest) or as references to an external secret provider.

Exactly one of `value` or `value_provider_id` must be provided.

### Schema

| Field | Type | Required | Computed | Mutable | Description |
|-------|------|----------|----------|---------|-------------|
| `id` | string | — | yes | — | UUID identifier |
| `organization_id` | string | yes | — | no (forces replacement) | Organization scope |
| `name` | string | yes | — | yes | Secret name |
| `value` | string | optional | — | yes | Direct secret value, encrypted at rest. Sensitive. Mutually exclusive with `value_provider_id` |
| `value_provider_id` | string | optional | — | yes | Reference to an `agyn_secret_provider`. Mutually exclusive with `value`. Requires `value_reference` |
| `value_reference` | string | optional | — | yes | Identifier of the secret in the external provider. Required when `value_provider_id` is set. For Vault: `<mount>/<path>/<key>` |

### Lifecycle

| Operation | API Method | Notes |
|-----------|------------|-------|
| **Create** | `CreateSecret` | Creates the secret with local or remote storage |
| **Read** | `GetSecret` | Reads by `id`. Does not return the plaintext `value` |
| **Update** | `UpdateSecret` | Updates `name`, `value`, `value_provider_id`, `value_reference` |
| **Delete** | `DeleteSecret` | Deletes the secret |
| **Import** | — | Import by `id` |

### Example (Local)

```hcl
resource "agyn_secret" "api_key" {
  organization_id = agyn_organization.eng.id
  name            = "openai-api-key"
  value           = var.openai_api_key
}

resource "agyn_env" "api_key" {
  agent_id  = agyn_agent.assistant.id
  name      = "OPENAI_API_KEY"
  secret_id = agyn_secret.api_key.id
}
```

### Example (Remote — Vault)

```hcl
resource "agyn_secret_provider" "vault" {
  organization_id = agyn_organization.eng.id
  type            = "vault"
  address         = "http://vault:8200"
  token           = var.vault_token
}

resource "agyn_secret" "api_key" {
  organization_id   = agyn_organization.eng.id
  name              = "openai-api-key"
  value_provider_id = agyn_secret_provider.vault.id
  value_reference   = "secret/platform/keys/openai_api_key"
}
```

## Provisioning Resources

Some resources produce a **service token** on creation. The token is returned only once (on `terraform apply`) and stored in Terraform state. It is used to enroll the service at startup. See [Authentication — Service Tokens](../authn.md#service-tokens).

| Resource | Token Output | Enrollment |
|----------|-------------|------------|
| `agyn_app` | `service_token` | App presents token at startup → receives OpenZiti identity |
| `agyn_runner` | `service_token` | Runner presents token at startup → receives OpenZiti identity |

## Agent Resource Structure

Agent resources (everything except `agyn_app`, `agyn_runner`, `agyn_user`, `agyn_secret_provider`, `agyn_secret`, and `agyn_membership`) share a common envelope:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID, computed |
| `description` | string | Human-readable description, optional |

Resource-specific fields are exposed as typed HCL attributes. The canonical schema for each resource is documented in [Resource Definitions](../resource-definitions.md).

### Ownership

Most sub-resources (MCP, Skill, Hook, ENV, InitScript) have an ownership field (`agent_id`, `mcp_id`, or `hook_id`) that determines which parent resource they belong to. These are required, immutable after creation, and expressed as standard Terraform resource attributes — not as attachment resources.
