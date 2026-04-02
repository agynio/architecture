# MCP Architecture

## Target

- [MCP](../architecture/mcp.md)
- [Resource Definitions — MCP](../architecture/resource-definitions.md#mcp)
- [agynd-cli](../architecture/agynd-cli.md)
- [Agent Init — Environment Variable Contract](../architecture/agent-init.md#environment-variable-contract)
- [Agents Orchestrator — Workload Spec Assembly](../architecture/agents-orchestrator.md#workload-spec-assembly)

## Acceptance Signal

- MCP proto has a `name` field; Agents service enforces uniqueness within agent.
- Sidecar proxy binary bridges stdio MCP servers to streamable HTTP, reading `MCP_PORT`.
- Orchestrator assigns ports to MCP sidecars and sets `MCP_PORT` on each sidecar and `AGENT_MCP_SERVERS` on the agent container.
- `agynd` parses `AGENT_MCP_SERVERS` and configures agent CLIs with MCP endpoint lists.
- Agent CLIs connect directly to MCP sidecars over streamable HTTP on localhost.
