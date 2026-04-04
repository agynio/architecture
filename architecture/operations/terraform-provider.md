# Terraform Provider

The **Agyn Terraform provider** is the recommended way to configure agents and their dependencies as code.

**Repository:** [`agynio/terraform-provider-agyn`](https://github.com/agynio/terraform-provider-agyn)

The provider connects to the [Gateway](../gateway.md) using a generated gRPC client from the gateway proto definitions. ConnectRPC's gRPC protocol support means the provider communicates with the same Gateway endpoint used by all other clients — no separate API surface.

## Resources

| Terraform Resource | API Resource | Description |
|-------------------|-------------|-------------|
| `agyn_app` | App | App registration (slug, name, icon). Returns service token for enrollment |
| `agyn_runner` | Runner | Runner registration (name, organization scope, labels). Returns service token for enrollment |
| `agyn_agent` | Agent | Agent definition (identity, model, image, compute resources, configuration) |
| `agyn_volume` | Volume | Volume definition (persistent/ephemeral, mount path, size) |
| `agyn_volume_attachment` | Volume Attachment | Relationship between a volume and a container (agent, MCP, or hook) |
| `agyn_image_pull_secret` | Image Pull Secret | Registry credential (registry hostname, username, password/token). Managed by the Secrets service |
| `agyn_image_pull_secret_attachment` | Image Pull Secret Attachment | Relationship between an image pull secret and a container (agent, MCP, or hook) |
| `agyn_mcp` | MCP | MCP server definition (image, command, compute resources) |
| `agyn_skill` | Skill | Skill definition (name, body) |
| `agyn_hook` | Hook | Hook definition (event, function, image, compute resources) |
| `agyn_env` | ENV | Environment variable (name, plain value or secret reference) |
| `agyn_init_script` | InitScript | Initialization script (shell script content) |
| `agyn_membership` | Membership | Organization membership (identity, organization, role). See [Organizations — Members Management](../organizations.md#members-management) |

## Provisioning Resources

Some resources produce a **service token** on creation. The token is returned only once (on `terraform apply`) and stored in Terraform state. It is used to enroll the service at startup. See [Authentication — Service Tokens](../authn.md#service-tokens).

| Resource | Token Output | Enrollment |
|----------|-------------|------------|
| `agyn_app` | `service_token` | App presents token at startup → receives OpenZiti identity |
| `agyn_runner` | `service_token` | Runner presents token at startup → receives OpenZiti identity |

## Agent Resource Structure

Agent resources (everything except `agyn_app`, `agyn_runner`, and `agyn_membership`) share a common envelope:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID, computed |
| `description` | string | Human-readable description, optional |

Resource-specific fields are exposed as typed HCL attributes. The canonical schema for each resource is documented in [Resource Definitions](../resource-definitions.md).

### Ownership

Most sub-resources (MCP, Skill, Hook, ENV, InitScript) have an ownership field (`agent_id`, `mcp_id`, or `hook_id`) that determines which parent resource they belong to. These are required, immutable after creation, and expressed as standard Terraform resource attributes — not as attachment resources.

