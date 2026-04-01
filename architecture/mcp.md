# MCP (Model Context Protocol)

## Overview

Agents use tools provided via [MCP](https://modelcontextprotocol.io/) (Model Context Protocol). MCP servers run as sidecar containers inside the agent pod, sharing the network namespace. Each MCP server exposes tools over streamable HTTP on a localhost port. The agent CLI connects to each MCP server directly — there is no aggregation proxy.

## Transports

MCP defines two standard transports. The platform supports both:

| Transport | Description | Usage |
|-----------|-------------|-------|
| **stdio** | Newline-delimited JSON-RPC 2.0 over stdin/stdout | MCP servers that only support stdio. Wrapped by the [sidecar proxy](#sidecar-proxy) to expose streamable HTTP |
| **Streamable HTTP** | JSON-RPC 2.0 over HTTP (SSE for server-to-client streaming) | MCP servers that natively support HTTP. Exposed directly on a localhost port — no proxy needed |

From the agent CLI's perspective, every MCP server is a streamable HTTP endpoint on `localhost:<port>`. The transport difference is handled at the sidecar level.

## Architecture

```mermaid
graph TB
    subgraph "Agent Pod (shared network namespace)"
        subgraph "Agent Container"
            AgentCLI[Agent CLI]
        end

        subgraph "MCP Sidecar: stdio server"
            Proxy[Sidecar Proxy]
            StdioMCP[MCP Server<br/>stdio]
            Proxy -->|stdin/stdout| StdioMCP
        end

        subgraph "MCP Sidecar: HTTP server"
            HttpMCP[MCP Server<br/>streamable HTTP]
        end
    end

    AgentCLI -->|streamable HTTP<br/>localhost:port| Proxy
    AgentCLI -->|streamable HTTP<br/>localhost:port| HttpMCP
```

### Streamable HTTP MCP servers

MCP servers that natively support streamable HTTP expose their port directly. The sidecar container runs the MCP server process as its entrypoint. No proxy is involved.

### stdio MCP servers

MCP servers that only support stdio cannot accept network connections. The sidecar container runs a **sidecar proxy** that:

1. Spawns the MCP server process as a child.
2. Communicates with it over stdin/stdout (JSON-RPC 2.0).
3. Exposes a streamable HTTP endpoint on a localhost port.
4. Bridges requests and responses between HTTP and stdio.

The sidecar proxy is a generic stdio-to-HTTP adapter. It has no MCP-semantic awareness beyond transport bridging.

## Sidecar Proxy

The sidecar proxy runs inside MCP sidecar containers that host stdio-only MCP servers. It is the sidecar container's entrypoint.

| Aspect | Detail |
|--------|--------|
| Role | Bridges stdio MCP servers to streamable HTTP |
| Input | MCP server process communicating over stdin/stdout |
| Output | Streamable HTTP endpoint on a localhost port |
| Lifecycle | Spawns the MCP server process, manages health checks, restarts with backoff on failure |

The proxy does not modify, inspect, or transform MCP messages beyond transport reframing (JSON-RPC 2.0 lines ↔ HTTP request/response bodies).

## Agent CLI Connection

The agent CLI connects to each MCP server independently over streamable HTTP. `agynd` configures the agent CLI with the list of MCP endpoints (one `localhost:<port>` per server) before spawning it. The agent CLI handles:

- Tool discovery (`tools/list`) from each server
- Tool call routing to the correct server
- Tool namespacing to prevent collisions across servers

Tool namespacing is owned by each agent CLI implementation. The platform does not enforce a namespacing convention.

## Configuration Flow

```mermaid
sequenceDiagram
    participant O as Orchestrator
    participant R as Runner
    participant D as agynd
    participant A as Agent CLI
    participant S1 as MCP Sidecar (stdio)
    participant S2 as MCP Sidecar (HTTP)

    O->>R: StartWorkload (MCP sidecars with port assignments)
    R->>S1: Start sidecar (proxy + stdio MCP server)
    R->>S2: Start sidecar (HTTP MCP server)
    D->>D: Read MCP config (server names + ports)
    D->>A: Configure MCP endpoints (localhost:port per server)
    D->>A: Spawn agent CLI
    A->>S1: tools/list (streamable HTTP)
    A->>S2: tools/list (streamable HTTP)
    A->>A: Discover and namespace tools
```

1. The [Agents Orchestrator](agents-orchestrator.md) assembles the workload spec with MCP sidecar definitions and port assignments.
2. The [Runner](runner.md) starts sidecar containers — each exposes streamable HTTP on its assigned port.
3. `agynd` reads the MCP configuration (server names and ports) and configures the agent CLI with the endpoint list.
4. The agent CLI connects to each endpoint, discovers tools, and begins using them.

## Resource Definition

MCP servers are defined as agent sub-resources. See [Resource Definitions — MCP](resource-definitions.md#mcp) for the schema. Each MCP resource specifies the container image and startup command. Environment variables, init scripts, and volumes are attached via their respective sub-resources.
