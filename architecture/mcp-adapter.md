# MCP Adapter

## Overview

The MCP Adapter is a standalone binary that wraps any MCP server and exposes a uniform gRPC interface. It is the entrypoint of each MCP server sidecar container within an [agent workload](orchestrator.md#agent-workload).

The adapter handles the protocol translation between the MCP server's native transport (stdio or Streamable HTTP) and the gRPC interface that the agent consumes on `localhost`.

## Architecture

```
Agent Workload (single pod)
├── Main container: agent
├── Sidecar: MCP server
│   ├── MCP Adapter binary (entrypoint)
│   │   ├── gRPC server (localhost:PORT)
│   │   └── MCP client (stdio or HTTP to local subprocess)
│   └── MCP server process (launched by adapter)
├── Sidecar: MCP server (another)
│   └── ...
└── Sidecar: OpenZiti tunnel
```

The adapter is added to any MCP server image and used as the sidecar container entrypoint. The MCP server image provides the runtime and dependencies (Node.js, Python, etc.). The adapter binary is a static executable with no runtime dependencies.

The agent and MCP sidecars share the pod's network namespace. The agent connects to each MCP server via `localhost` on distinct gRPC ports. No OpenZiti or external network is involved in agent-to-MCP communication.

## Transport Modes

The adapter supports two modes, matching the two standard MCP transports.

### stdio Mode

The adapter launches the MCP server as a subprocess and communicates via stdin/stdout (newline-delimited JSON-RPC 2.0).

```
adapter --mode stdio --command "npx best-mcp-server" --grpc-port 50051
```

This supports the majority of existing MCP servers, which are distributed via package managers (`npx`, `uvx`, `pip`, etc.) and implement only the stdio transport.

### HTTP Mode

The adapter launches the MCP server as a subprocess and connects to it via Streamable HTTP (HTTP POST/GET + optional SSE) on a local port.

```
adapter --mode http --command "uvx some-mcp --port 8080" --http-port 8080 --grpc-port 50051
```

The adapter waits for the MCP server to become ready on the specified port before accepting gRPC connections.

## Lifecycle

The adapter owns the MCP server process lifecycle:

1. **Start** — launch the MCP server subprocess with the configured command.
2. **Initialize** — perform the MCP `initialize` handshake (capability exchange).
3. **Ready** — begin accepting gRPC connections from the agent.
4. **Health check** — periodic heartbeat to detect MCP server failures.
5. **Restart** — if the MCP server process dies, restart it with configurable backoff.
6. **Shutdown** — on SIGTERM, gracefully stop the MCP server process and drain gRPC connections.

## gRPC Interface

The adapter exposes a gRPC service that mirrors the MCP protocol operations. The agent calls these RPCs on `localhost`.

The gRPC proto is defined in `agynio/api`. Key operations:

| RPC | MCP Method | Description |
|-----|-----------|-------------|
| `ListTools` | `tools/list` | List available tools |
| `CallTool` | `tools/call` | Execute a tool and return the result |
| `ListResources` | `resources/list` | List available resources |
| `ReadResource` | `resources/read` | Read a resource |
| `ListPrompts` | `prompts/list` | List available prompts |
| `GetPrompt` | `prompts/get` | Get a prompt |

The adapter translates between gRPC request/response types and MCP JSON-RPC messages. Streaming tool results are supported via server-streaming RPCs.

## Configuration

The adapter is configured entirely via command-line flags and environment variables. No config files.

| Flag | Env | Required | Description |
|------|-----|----------|-------------|
| `--mode` | `MCP_ADAPTER_MODE` | Yes | `stdio` or `http` |
| `--command` | `MCP_ADAPTER_COMMAND` | Yes | Shell command to launch the MCP server |
| `--grpc-port` | `MCP_ADAPTER_GRPC_PORT` | Yes | Port for the gRPC server |
| `--http-port` | `MCP_ADAPTER_HTTP_PORT` | No (http mode) | Port the MCP server listens on (http mode only) |
| `--startup-timeout` | `MCP_ADAPTER_STARTUP_TIMEOUT` | No | Max time to wait for MCP server readiness |
| `--restart-max-attempts` | `MCP_ADAPTER_RESTART_MAX_ATTEMPTS` | No | Max restart attempts on failure |
| `--restart-backoff` | `MCP_ADAPTER_RESTART_BACKOFF` | No | Base backoff between restarts |
| `--heartbeat-interval` | `MCP_ADAPTER_HEARTBEAT_INTERVAL` | No | Heartbeat interval for health checks |
