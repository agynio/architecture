# files-mcp

## Overview

`files-mcp` is a platform-provided MCP server that gives agents access to files uploaded to the platform. It exposes a single `read_file` tool that fetches file content from the [Files](media.md) service and returns it in the appropriate MCP content type. The LLM calls this tool when it encounters `agyn://file/<id>` references in the conversation context.

| Aspect | Details |
|--------|---------|
| Repository | `agynio/files-mcp` |
| Language | Go |
| Transport | Streamable HTTP |
| Role | MCP server — file access for agents |

## Architecture

```mermaid
graph TB
    subgraph "Agent Pod (shared network namespace)"
        subgraph "Agent Container"
            AgentCLI[Agent CLI]
        end

        subgraph "MCP Sidecar: files-mcp"
            FilesMCP[files-mcp<br/>streamable HTTP]
        end

        subgraph "Ziti Sidecar"
            Ziti[Ziti Sidecar]
        end
    end

    subgraph Platform
        Gateway
        Files[Files Service]
    end

    AgentCLI -->|streamable HTTP<br/>localhost:port| FilesMCP
    FilesMCP -->|GetFileMetadata, GetFileContent<br/>via gateway.ziti| Gateway
    Gateway --> Files
    Ziti -.->|OpenZiti mTLS| Gateway
```

`files-mcp` runs as a sidecar container in the agent pod. It connects to the [Gateway](gateway.md) via the `gateway.ziti` OpenZiti hostname (transparently intercepted by the pod's Ziti sidecar) to access the Files service API. File content is downloaded through the Files service's `GetFileContent` RPC — the MCP server does not access object storage directly.

## Tool

### `read_file`

Reads a file from the platform's Files service and returns its content.

**Tool definition:**

```json
{
  "name": "read_file",
  "description": "Read a file from the platform. Use this tool to access file content when you see agyn://file/ references in the conversation.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "file_id": {
        "type": "string",
        "description": "The file ID from an agyn://file/ URI (the part after agyn://file/)"
      }
    },
    "required": ["file_id"]
  }
}
```

### Flow

```mermaid
sequenceDiagram
    participant A as Agent CLI
    participant MCP as files-mcp
    participant GW as Gateway (via Ziti)
    participant F as Files Service

    A->>MCP: tools/call read_file {file_id}
    MCP->>GW: GetFileMetadata(file_id)
    GW->>F: GetFileMetadata(file_id)
    F-->>GW: {id, filename, content_type, size_bytes}
    GW-->>MCP: File metadata
    MCP->>GW: GetFileContent(file_id)
    GW->>F: GetFileContent(file_id)
    F-->>GW: stream of chunks
    GW-->>MCP: File bytes
    MCP->>MCP: Build MCP content based on content_type
    MCP-->>A: tools/call result
```

1. Agent CLI sends `tools/call` with `read_file` and a `file_id`.
2. MCP server fetches file metadata from the Files service (via Gateway) to determine `content_type`, `filename`, and `size_bytes`.
3. MCP server checks `size_bytes` against `MAX_FILE_SIZE`. If exceeded, returns an error without downloading.
4. MCP server calls `GetFileContent` on the Files service (via Gateway) to stream the file content.
5. MCP server builds the appropriate MCP tool result content based on the file's MIME type.
6. MCP server returns the tool result to the agent CLI.

### Response Content Types

The MCP server selects the response content type based on the file's MIME type:

| File MIME type | MCP content type | Format |
|----------------|-----------------|--------|
| `image/*` | `image` | `{ "type": "image", "data": "<base64>", "mimeType": "<content_type>" }` |
| `text/*`, `application/json`, `application/xml`, `application/yaml` | `text` | `{ "type": "text", "text": "<decoded file content>" }` |
| All other types | `resource` (binary) | `{ "type": "resource", "resource": { "uri": "agyn://file/<file_id>", "mimeType": "<content_type>", "blob": "<base64>" } }` |

**Examples:**

Image file (`image/png`):
```json
{
  "content": [
    {
      "type": "image",
      "data": "iVBORw0KGgo...",
      "mimeType": "image/png"
    }
  ]
}
```

Text file (`text/plain`):
```json
{
  "content": [
    {
      "type": "text",
      "text": "file content here..."
    }
  ]
}
```

Binary file (`application/pdf`):
```json
{
  "content": [
    {
      "type": "resource",
      "resource": {
        "uri": "agyn://file/file-uuid",
        "mimeType": "application/pdf",
        "blob": "JVBERi0xLjQ..."
      }
    }
  ]
}
```

### Error Handling

Errors are reported as MCP tool execution errors (`isError: true`), not protocol-level errors:

| Condition | Error behavior |
|-----------|---------------|
| File not found | `isError: true`, text: "file not found: `<file_id>`" |
| File exceeds max size | `isError: true`, text: "file too large: `<size_bytes>` bytes exceeds limit of `<max_bytes>` bytes" |
| Download failure | `isError: true`, text: "failed to download file: `<details>`" |
| Gateway/Files service unavailable | `isError: true`, text: "platform service unavailable" |

## Authentication

`files-mcp` authenticates to the Gateway using the same mechanisms available to all components in the agent pod:

| Method | Mechanism | Use Case |
|--------|-----------|----------|
| **Network identity (Ziti sidecar)** | Pod-level [OpenZiti](authn.md#network-identity-openziti) mTLS via the Ziti sidecar — automatic when the sidecar is present | Primary. Inside agent pods on the platform |
| **API token** | `AGYN_API_TOKEN` environment variable, sent as `Authorization: Bearer` to the Gateway | Local development, testing, CI |

The API token fallback allows running `files-mcp` outside the platform for testing. When `AGYN_API_TOKEN` is set, the MCP server uses it for Gateway authentication instead of relying on the Ziti sidecar.

## Configuration

| Variable | Required | Description |
|----------|----------|-------------|
| `MCP_PORT` | yes | Port to listen on (assigned by the [Agents Orchestrator](agents-orchestrator.md)) |
| `GATEWAY_ADDRESS` | yes | Gateway address. Inside the platform: `gateway.ziti` (intercepted by Ziti sidecar). Outside: `https://gateway.agyn.dev` |
| `AGYN_API_TOKEN` | no | API token for Gateway authentication. Used when Ziti sidecar is not present (local dev, testing) |
| `MAX_FILE_SIZE` | no | Maximum file size in bytes the tool will read. Default: configurable per deployment |

## Deployment

`files-mcp` is deployed as an MCP sidecar container in the agent pod, following the standard [MCP sidecar pattern](mcp.md). It is a streamable HTTP server — no sidecar proxy is needed.

To add file access to an agent, create an [MCP resource](resource-definitions.md#mcp) on the agent definition pointing to the `files-mcp` image. The [Agents Orchestrator](agents-orchestrator.md) assembles it into the agent pod as a sidecar, the same as any other MCP server.

| Aspect | Detail |
|--------|--------|
| Image | `ghcr.io/agynio/files-mcp:latest` |
| Transport | Streamable HTTP (native) |
| Port | Assigned by Orchestrator via `MCP_PORT` |
| Proxy | Not needed — native streamable HTTP |

## MCP Capabilities

```json
{
  "capabilities": {
    "tools": {}
  }
}
```

The server declares only the `tools` capability. It does not support `listChanged` (the tool list is static), resources, or prompts.
