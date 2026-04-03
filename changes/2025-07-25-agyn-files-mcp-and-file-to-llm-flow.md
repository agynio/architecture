# agyn-files-mcp and File-to-LLM Flow

## Target

- [Media](../architecture/media.md)
- [agyn-files-mcp](../architecture/agyn-files-mcp.md)
- [Agent Implementation — MCP-to-LLM Translation](../architecture/agent/implementation.md#mcp-to-llm-translation)
- [Agent Implementation — Message Formatting](../architecture/agent/implementation.md#message-formatting)
- [Agent — Agent Contract](../architecture/agent/overview.md#agent-contract)

## Delta

The `agyn-files-mcp` MCP server (`agynio/agyn-files-mcp`) does not exist. The agent implementation does not yet format messages with `agynfile://` URIs or translate MCP tool results to OpenAI Responses API format. The following are needed:

- **agyn-files-mcp service**: new Go MCP server (streamable HTTP) exposing a `read_file` tool. Reads file metadata and content from the Files service via Gateway, returns appropriate MCP content types (text, image, resource) based on MIME type. Runs as a sidecar in agent pods.
- **Message formatting in `agn`**: Load stage must append `agynfile://<id>` URIs to message text for messages with file attachments.
- **MCP-to-LLM translation in `agn`**: CallTools stage must translate MCP tool result content types (text, image, audio, resource) to OpenAI Responses API function_call_output content types (input_text, input_image, input_file).
- **Agents Orchestrator**: automatically include `agyn-files-mcp` as a sidecar for agents with file access enabled.

## Acceptance Signal

- An agent can receive a message with file attachments, the LLM sees `agynfile://` references, calls `read_file`, and receives file content in the correct format for its provider API.
- The MCP-to-LLM translation handles all MCP content types (text, image, audio, resource with text, resource with blob).
- `agyn-files-mcp` authenticates via Ziti sidecar in production and via API token in development/testing.

## Notes

- The `read_file` tool uses a lazy, on-demand approach — file content is only fetched when the LLM explicitly requests it.
- `GetDownloadUrl` on the Files service is retained — it is now consumed by `agyn-files-mcp` instead of directly by the agent.
- The MCP-to-LLM translation is generic and applies to all MCP tool results, not just file reads.
