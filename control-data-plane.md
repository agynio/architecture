# Control Plane & Data Plane

## Definitions

**Control plane** manages the desired state of the system. It is stateless and responsible for reconciliation of dynamic resources (agents, channels, workspaces, etc.). It answers the question: *"what should be running?"*

**Data plane** handles the actual work: carrying messages, executing agent workloads, streaming events. It answers the question: *"how does the work get done?"*

## Criteria for Classification

| Criterion | Control Plane | Data Plane |
|-----------|--------------|------------|
| **State ownership** | Manages desired state declarations; does not hold runtime data | Manages and processes live data (messages, streams, agent context) |
| **Statelessness** | Stateless — can be restarted without losing runtime progress | May be stateful — holds connections, sessions, in-flight work |
| **Reconciliation** | Runs reconciliation loops to converge actual → desired state | Does not reconcile — executes what it is told |
| **Scaling model** | Scales with number of resource definitions | Scales with traffic / workload volume |
| **Failure impact** | Temporary loss delays new deployments/changes; existing workloads continue | Temporary loss disrupts active work (messages, agent runs) |

## Current Service Mapping

| Service | Plane | Rationale |
|---------|-------|-----------|
| **Teams** | Control | Manages desired state of team resources (agent definitions, MCP server configs, workspace configs) |
| **Agents** (orchestrator) | Control | Decides which agent workloads should exist based on pending messages; reconciles agent lifecycle |
| **Channels** (configuration) | Control | Defines channel desired state (e.g., Slack token + channel ID). Control plane reconciles channel connections |
| **Channels** (connection) | Data | Maintains live connections to 3rd-party APIs, translates messages bidirectionally |
| **Threads** | Data | Carries conversation messages between participants |
| **Notifications** | Data | Holds persistent socket connections, fans out real-time events |
| **Gateway** | Data | Routes external API requests to internal services |
| **Agent-State** | Data | Stores and retrieves agent conversation context |
| **Tracing** | Data | Ingests and serves tracing data |
| **Runner** | *Split needed* | See below |

## Runner: Split Needed

The Runner currently combines two concerns:

1. **Workload scheduling and lifecycle** (control plane): Deciding which containers to start/stop, managing labels, TTL, cleanup.
2. **Workload execution** (data plane): Running exec commands, streaming logs, providing the compute surface.

Moving forward, the Runner should be split:
- **Runner Controller** (control plane): Manages workload desired state, reconciles container/pod lifecycle.
- **Runner Worker** (data plane): Provides exec, log streaming, and archive capabilities for running workloads.

## Reconciliation

The control plane should implement a reconciliation loop that continuously converges actual state toward desired state. This is a migration from the current event-driven approach.

Key resources to reconcile:
- **Agents**: Ensure agent workloads exist for threads with pending messages; remove idle agents.
- **Channels**: Ensure channel connections match their configuration (e.g., reconnect on token rotation).
- **Workspaces**: Ensure workspace containers match desired image/config; enforce TTL.

The reconciliation approach (CRDs + operators vs. custom loop) is an [open question](open-questions.md#reconciliation-approach).
