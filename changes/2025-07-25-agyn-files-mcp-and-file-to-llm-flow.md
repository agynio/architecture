# agyn-files-mcp and File-to-LLM Flow

## Target

- [Media](../architecture/media.md)
- [agyn-files-mcp](../architecture/agyn-files-mcp.md)
- [agynd — Message Formatting](../architecture/agynd-cli.md#message-formatting)
- [Agent Implementation — MCP-to-LLM Translation](../architecture/agent/implementation.md#mcp-to-llm-translation)
- [Agent — Agent Contract](../architecture/agent/overview.md#agent-contract)

## Delta

The `agyn-files-mcp` MCP server (`agynio/agyn-files-mcp`) does not exist. `agynd` does not yet format thread messages with `agyn://file/` URIs. The agent implementation does not yet translate MCP tool results to OpenAI Responses API format. The following are needed:

- **agyn-files-mcp service**: new Go MCP server (streamable HTTP) exposing a `read_file` tool. Reads file metadata and content from the Files service via Gateway, returns appropriate MCP content types (text, image, resource) based on MIME type. Runs as a sidecar in agent pods.
- **Message formatting in `agynd`**: when translating thread messages for the agent CLI, `agynd` appends `agyn://file/<id>` URIs to the message text for messages with file attachments. The agent CLI receives pre-formatted plain text with no knowledge of the underlying thread message structure.
- **MCP-to-LLM translation in `agn`**: CallTools stage must translate MCP tool result content types (text, image, audio, resource) to OpenAI Responses API function_call_output content types (input_text, input_image, input_file).
- **Agents Orchestrator**: automatically include `agyn-files-mcp` as a sidecar for agents with file access enabled.

## Acceptance Signal

- `agynd` translates thread messages with file attachments into plain text with `agyn://file/` URIs before feeding them to the agent CLI.
- An agent can receive a message with file references, the LLM sees `agyn://file/` references, calls `read_file`, and receives file content in the correct format for its provider API.
- The MCP-to-LLM translation handles all MCP content types (text, image, audio, resource with text, resource with blob).
- `agyn-files-mcp` authenticates via Ziti sidecar in production and via API token in development/testing.

## Notes

- The `read_file` tool uses a lazy, on-demand approach — file content is only fetched when the LLM explicitly requests it.
- `GetDownloadUrl` on the Files service is retained — it is now consumed by `agyn-files-mcp` instead of directly by the agent.
- The MCP-to-LLM translation is generic and applies to all MCP tool results, not just file reads.
- The responsibility boundary is clear: `agynd` owns thread message translation (platform protocol), `agn` owns MCP-to-LLM content type translation (LLM protocol).
