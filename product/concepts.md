# Concepts

Domain glossary — canonical definitions of product terms. Both product and architecture documentation reference these definitions.

| Term | Definition |
|------|-----------|
| **Agent** | An AI entity configured with an LLM model, tools, and instructions. Agents receive messages, reason, execute tools, and respond within conversations. |
| **Agent Availability** | Per-agent setting that controls who may initiate a conversation with the agent. `internal` lets any organization member start a conversation; `private` restricts initiation to identities that hold an agent role. Availability does not affect whether the agent is visible in organization lists. |
| **Agent Role** | A grant from an identity to a specific agent. Three roles: `owner` (manage roles, change availability, delete, edit and read configuration, start conversations), `maintainer` (edit and read configuration, start conversations), and `participant` (start conversations only). Organization owners hold owner-level capabilities on every agent in their organization. |
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
