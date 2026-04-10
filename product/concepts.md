# Concepts

Domain glossary — canonical definitions of product terms. Both product and architecture documentation reference these definitions.

| Term | Definition |
|------|-----------|
| **Agent** | An AI entity configured with an LLM model, tools, and instructions. Agents receive messages, reason, execute tools, and respond within conversations. |
| **Chat** | The platform's communication interface. Users create conversations with any combination of users and agents, all managed in a single list-detail view. |
| **Container** | A running workload (Kubernetes pod) attached to a conversation's agent. Containers provide terminal access for inspection. |
| **Context** | The set of items (messages, tool results, memory, summaries) assembled into a prompt for an LLM call. Context is paginated and inspectable per LLM event. |
| **Conversation** | A persistent exchange between participants (users, agents, or both). Conversations have a lifecycle (open → resolved). |
| **Gateway** | The external API surface of the platform. All client applications communicate with the platform through the Gateway via ConnectRPC. |
| **Identity** | A unique entity in the platform — a user, an agent, or a service. Every identity has a type and a platform-wide ID. |
| **MCP Server** | A Model Context Protocol server that exposes tools to agents over a secure network. Agents connect to MCP servers to execute actions in external systems. |
| **Notification** | A real-time event delivered to the UI via WebSocket. Notifications drive live updates for conversations, messages, runs, and tool output. |
| **Organization** | A multi-tenant boundary grouping users, agents, and configuration resources. Access control is scoped to organizations. |
| **Reminder** | A scheduled follow-up attached to a conversation, created by an agent. Reminders notify the user at a specified time and can be cancelled. |
| **Run** | A single execution cycle within a conversation, triggered when the agent processes unacknowledged messages. A conversation accumulates multiple runs over its lifetime. |
| **Run Event** | A discrete step within a run: a message received, an LLM call made, a tool executed, or a context summarization performed. Events are the atomic unit of observability. |
| **Summarization** | An event where the agent's context is compressed to stay within token limits. The old context is replaced with a shorter summary. |
| **Tool** | A capability available to an agent via an MCP server. Tools accept structured input, execute an action, and return output. Tool executions produce stdout/stderr streams. |

## Related architecture

- [Identity](../architecture/identity.md)
- [Organizations](../architecture/organizations.md)
- [Agents service](../architecture/agents-service.md)
- [Threads](../architecture/threads.md)
- [Notifications](../architecture/notifications.md)
- [Apps](../architecture/apps.md)
- [MCP](../architecture/mcp.md)
- [Resource definitions](../architecture/resource-definitions.md)
