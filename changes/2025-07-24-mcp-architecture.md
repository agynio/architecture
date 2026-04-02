# MCP Architecture

## Target

- [MCP](../architecture/mcp.md)
- [Resource Definitions — MCP](../architecture/resource-definitions.md#mcp)
- [agynd-cli](../architecture/agynd-cli.md)
- [Agent Overview — Tools](../architecture/agent/overview.md#tools)
- [Agent Init — Environment Variable Contract](../architecture/agent-init.md#environment-variable-contract)
- [Agents Orchestrator — Workload Spec Assembly](../architecture/agents-orchestrator.md#workload-spec-assembly)

## Delta

- The MCP sidecar proxy (stdio-to-streamable-HTTP bridge) does not exist. Needs to be implemented as a standalone binary that MCP sidecar containers use as their entrypoint for stdio-only MCP servers.
- `agynd` still contains the aggregated MCP proxy implementation. It should be replaced with MCP endpoint configuration — parsing `AGENT_MCP_SERVERS` env var and writing MCP server entries into the agent CLI config.
- The MCP resource in the Agents service does not have a `name` field. Needs to be added to the proto definition and stored in PostgreSQL (unique within agent).
- The Orchestrator does not assign MCP ports or set the `MCP_PORT` / `AGENT_MCP_SERVERS` environment variables. Port allocation logic needs to be added to workload spec assembly.

## Acceptance Signal

- MCP proto has a `name` field; Agents service enforces uniqueness within agent.
- A sidecar proxy binary exists that bridges stdio MCP servers to streamable HTTP, reading `MCP_PORT` for its listen port.
- The Orchestrator assigns ports to MCP sidecars and sets `MCP_PORT` on each sidecar and `AGENT_MCP_SERVERS` on the agent container.
- `agynd` parses `AGENT_MCP_SERVERS` and configures agent CLIs with MCP endpoint lists (no aggregation proxy).
- Agent CLIs connect directly to MCP sidecars over streamable HTTP on localhost.
- Both stdio and streamable HTTP MCP servers work end-to-end in agent pods.

## Notes

- Streamable HTTP MCP servers require no proxy — the sidecar container runs the MCP server process directly.
- Tool namespacing is owned by each agent CLI implementation, not by the platform.
- Port allocation starts from a configurable base port (default: 8100) and increments per MCP server, sorted by stable key (id).
