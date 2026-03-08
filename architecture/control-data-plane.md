# Control Plane & Data Plane

## Definitions

**Control plane** manages the desired state of the system. It is stateless and responsible for reconciliation of dynamic resources (agents, channels, workspaces, etc.). It answers: *"what should be running?"*

**Data plane** handles the actual work: carrying messages, executing agent workloads, streaming events. It answers: *"how does the work get done?"*

## Classification Criteria

| Criterion | Control Plane | Data Plane |
|-----------|--------------|------------|
| **State ownership** | Manages desired-state declarations; no runtime data | Processes live data (messages, streams, agent context) |
| **Statelessness** | Stateless — restartable without losing runtime progress | May be stateful — holds connections, sessions, in-flight work |
| **Reconciliation** | Runs reconciliation loops (actual → desired) | Does not reconcile — executes what it is told |
| **Scaling model** | Scales with number of resource definitions | Scales with traffic / workload volume |
| **Failure impact** | Temporary loss delays new changes; existing workloads continue | Temporary loss disrupts active work |

## Service Classification

```mermaid
graph LR
    subgraph Control Plane
        Teams
        AgentsOrch[Agents Orchestrator]
    end

    subgraph Data Plane
        Gateway
        Channels
        Threads
        Notifications
        Runner
        AgentState[Agent State]
        Tracing
    end

    Teams -->|desired state| AgentsOrch
    AgentsOrch -->|schedule workloads| Runner
    AgentsOrch -->|read pending messages| Threads
```

| Service | Plane | Rationale |
|---------|-------|-----------|
| **Teams** | Control | Manages desired state of team resources (agent definitions, MCP server configs, workspace configs) |
| **Agents orchestrator** | Control | Decides which agent workloads should exist; reconciles agent lifecycle |
| **Channels** (configuration) | Control | Defines channel desired state (credentials, target IDs, routing rules) |
| **Channels** (connection) | Data | Maintains live connections to 3rd-party APIs, translates messages |
| **Threads** | Data | Carries conversation messages between participants |
| **Notifications** | Data | Holds persistent connections, fans out real-time events |
| **Gateway** | Data | Routes external API requests to internal services |
| **Agent State** | Data | Stores and retrieves agent conversation context |
| **Tracing** | Data | Ingests and serves tracing data |
| **Runner** | Data | Executes workloads (containers/pods), provides exec and log streaming |

## Reconciliation

The control plane should implement a reconciliation loop that continuously converges actual state toward desired state.

Key resources to reconcile:
- **Agents** — Ensure agent workloads exist for threads with pending messages; remove idle agents.
- **Channels** — Ensure channel connections match their configuration (reconnect on credential rotation).
- **Workspaces** — Ensure workspace containers match desired image/config; enforce TTL.

The reconciliation approach (CRDs + operators vs. custom loop) is an [open question](../open-questions.md#reconciliation-approach).
