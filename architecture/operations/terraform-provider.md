# Terraform Provider

The **Agyn Terraform provider** is the recommended way to configure agents and their dependencies as code.

**Repository:** [`agynio/terraform-provider-agyn`](https://github.com/agynio/terraform-provider-agyn)

The provider connects to the [Gateway](../gateway.md) using a generated gRPC client from the gateway proto definitions. ConnectRPC's gRPC protocol support means the provider communicates with the same Gateway endpoint used by all other clients — no separate API surface.

## Resources

| Terraform Resource | API Resource | Description |
|-------------------|-------------|-------------|
| `agyn_agent` | Agent | Agent definition (identity, model, image, compute resources, configuration) |
| `agyn_volume` | Volume | Volume definition (persistent/ephemeral, mount path, size) |
| `agyn_volume_attachment` | Volume Attachment | Relationship between a volume and a container (agent, MCP, or hook) |
| `agyn_mcp` | MCP | MCP server definition (image, command, compute resources) |
| `agyn_skill` | Skill | Skill definition (name, body) |
| `agyn_hook` | Hook | Hook definition (event, function, image, compute resources) |
| `agyn_env` | ENV | Environment variable (name, plain value or secret reference) |
| `agyn_init_script` | InitScript | Initialization script (shell script content) |

## Resource Structure

All resources (except `agyn_volume_attachment`) share a common envelope:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID, computed |
| `description` | string | Human-readable description, optional |

Resource-specific fields are exposed as typed HCL attributes. The canonical schema for each resource is documented in [Resource Definitions](../resource-definitions.md).

### Ownership

Most sub-resources (MCP, Skill, Hook, ENV, InitScript) have an ownership field (`agent_id`, `mcp_id`, or `hook_id`) that determines which parent resource they belong to. These are required, immutable after creation, and expressed as standard Terraform resource attributes — not as attachment resources.

Volume is the exception: volumes are standalone resources connected to containers (agents, MCPs, or hooks) via `agyn_volume_attachment`, because volumes are reusable infrastructure that may outlive individual agents.
