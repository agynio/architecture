# MCP Architecture

## Target

- [MCP](../architecture/mcp.md)
- [agynd-cli](../architecture/agynd-cli.md)
- [Agent Overview — Tools](../architecture/agent/overview.md#tools)

## Delta

- The MCP sidecar proxy (stdio-to-streamable-HTTP bridge) does not exist. Needs to be implemented as a standalone binary that MCP sidecar containers use as their entrypoint for stdio-only MCP servers.
- `agynd` still contains the aggregated MCP proxy implementation. It should be replaced with MCP endpoint configuration — writing MCP server entries (name + `localhost:<port>`) into the agent CLI config instead of proxying tool calls.
- MCP resource definition does not include a `port` field or transport type indicator. The Orchestrator needs a way to assign ports to MCP sidecars and communicate whether the sidecar needs the stdio proxy.

## Acceptance Signal

- A sidecar proxy binary exists that bridges stdio MCP servers to streamable HTTP.
- `agynd` configures agent CLIs with MCP endpoint lists (no aggregation proxy).
- Agent CLIs connect directly to MCP sidecars over streamable HTTP on localhost.
- Both stdio and streamable HTTP MCP servers work end-to-end in agent pods.

## Notes

- Streamable HTTP MCP servers require no proxy — the sidecar container runs the MCP server process directly.
- Tool namespacing is owned by each agent CLI implementation, not by the platform.
