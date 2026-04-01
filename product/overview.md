# Overview

## Vision

Agyn is a platform for running AI agents that collaborate with humans through conversation. Users interact with agents via conversations — persistent exchanges where agents reason, use tools, and produce results. The platform handles agent lifecycle, tool connectivity, observability, and multi-tenant isolation so that teams can deploy and operate agents without building infrastructure.

## Value Proposition

- **Conversational agent interaction** — users work with agents through a familiar messaging interface, not dashboards or pipelines.
- **Full execution observability** — every LLM call, tool execution, and context decision is recorded and inspectable in real time.
- **Managed agent lifecycle** — the platform provisions, schedules, and tears down agent workloads automatically.
- **Tool connectivity via MCP** — agents access external tools through Model Context Protocol servers with secure networking.
- **Multi-tenant by default** — organizations, agents, and data are isolated through relationship-based access control.

## Target Users

| Persona | Primary Goal |
|---------|-------------|
| **End user** | Communicate with users and agents to delegate tasks, review results, and manage ongoing work |
| **Agent operator** | Monitor agent execution, diagnose failures, understand cost via run timelines and tracing |
| **Platform administrator** | Configure organizations, agents, tool servers, secrets, and access policies |

## Product Principles

- **Conversation is the interface.** Every interaction starts and ends in a conversation. The platform does not expose workflow builders, DAG editors, or batch job UIs.
- **Observability is not optional.** Every agent run produces a complete, inspectable trace. Users should never wonder "what did the agent do?"
- **Desired state, not procedures.** Product documentation describes the target experience. Implementation catches up; the spec does not describe the gap.
