# Terraform Provider

The **Agyn Terraform provider** is the recommended way to configure teams, agents, MCP servers, workspaces, and their relationships as code.

**Repository:** [`agynio/terraform-provider-agyn`](https://github.com/agynio/terraform-provider-agyn)

## Provider Configuration

```hcl
provider "agyn" {
  api_url = "https://agyn.dev:2496"  # Platform API base URL
}
```

Authentication is handled via the provider configuration (API key or token — see provider docs).

## Resources

| Resource | Description |
|----------|-------------|
| `agyn_agent` | Agent definition (model, behavior, prompts) |
| `agyn_mcp_server` | MCP server definition (command, namespace, env, timeouts) |
| `agyn_workspace_configuration` | Workspace container config (image, resources, TTL, env) |
| `agyn_memory_bucket` | Memory bucket (scope, collection prefix) |
| `agyn_tool` | Legacy tool definition (being replaced by MCP servers) |
| `agyn_attachment` | Typed relationship between resources (e.g., MCP server → agent) |

## Resource Structure

All resources (except `agyn_attachment`) follow a common envelope:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID, computed |
| `title` | string | Human-readable title, optional |
| `description` | string | Human-readable description, optional |
| `config` | string (JSON) | Resource-specific configuration |

The `config` field is currently an opaque JSON blob (`jsonencode({...})`). The typed schema for each resource's config is documented in [Resource Definitions](../resource-definitions.md).

### Attachment Structure

Attachments link resources together:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | UUID, computed |
| `kind` | string | Relationship type (e.g., `tool_agent`, `mcp_agent`) |
| `source_id` | string | Source resource UUID |
| `source_type` | string | Source entity type, computed |
| `target_id` | string | Target resource UUID |
| `target_type` | string | Target entity type, computed |

## Example

```hcl
resource "agyn_workspace_configuration" "dev" {
  title       = "Development Workspace"
  description = "Standard development container"
  config = jsonencode({
    image       = "ghcr.io/agynio/workspace:latest"
    cpu_limit   = "2"
    memory_limit = "4Gi"
    ttlSeconds  = 86400
    volumes = {
      enabled   = true
      mountPath = "/workspace"
    }
  })
}

resource "agyn_mcp_server" "shell" {
  title       = "Shell"
  description = "Shell command execution"
  config = jsonencode({
    namespace = "shell"
    command   = "mcp start --stdio"
  })
}

resource "agyn_agent" "developer" {
  title       = "Developer Agent"
  description = "Coding assistant"
  config = jsonencode({
    model        = "gpt-5"
    systemPrompt = "You are a helpful coding assistant."
    whenBusy     = "injectAfterTools"
    processBuffer = "allTogether"
  })
}

resource "agyn_attachment" "shell_to_agent" {
  kind      = "tool_agent"
  source_id = agyn_mcp_server.shell.id
  target_id = agyn_agent.developer.id
}

resource "agyn_attachment" "workspace_to_agent" {
  kind      = "tool_agent"
  source_id = agyn_workspace_configuration.dev.id
  target_id = agyn_agent.developer.id
}
```

## Known Limitation: Untyped Config

The `config` field in the Terraform provider is an opaque JSON string. The provider performs no schema validation beyond checking for valid JSON. This means:

- No autocompletion or type checking in HCL.
- Invalid field names or types are accepted by Terraform but rejected at runtime by the platform.
- Drift detection compares raw JSON strings.

This is a known issue. The target state is to have the provider expose typed attributes for each resource, validated at plan time. The canonical schema for each resource is documented in [Resource Definitions](../resource-definitions.md).
