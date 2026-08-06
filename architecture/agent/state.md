# Agent State

## Overview

Agent state is managed entirely by each agent implementation and persisted locally on disk. The agent decides what to store and how to organize it. There is no platform-level state service, schema, or contract for agent state.

Whether that disk **survives a workload stop** is a property of the [Environment](../resource-definitions.md#environment) the agent runs in, not a guarantee the platform makes. An environment declaring a persistent [Volume](../resource-definitions.md#volume) gives its agents memory across restarts; an environment declaring none gives them a fresh disk on every start. Neither is a malfunction — persistence is configuration, and different agents want different answers.

## Design Principles

- **Agent-owned.** Each agent implementation (our [`agn`](../agn-cli.md), wrapped Codex, wrapped Claude, custom agents) defines its own state format and storage layout. The platform does not interpret or validate agent state.
- **Disk-based.** State is written to a filesystem path. No external service dependency for state persistence.
- **Instance-scoped.** State is keyed by [agent instance](../agent-instances.md), not by thread or by class. Multiple threads that route to the same instance share the same state; different instances of the same class have independent state — and separate disks, even when they run the same environment.
- **Environment-provided lifetime.** State lives as long as the disk under it. A persistent volume survives workload stops and is released when the instance is deleted (subject to volume TTL); with no persistent volume, state lives exactly as long as the workload.

## How It Works

The [Agents Orchestrator](../agents-orchestrator.md) assembles the workload spec from the environment's [Volume](../resource-definitions.md#volume) definitions, provisioning one disk per definition per owner — keyed by `agent_instance_id` in the [Runners](../runners.md#volume-resource) records (distinct from the runner-local `instance_id`, which is the PVC name). The [Runner](../runner.md) creates PersistentVolumeClaims on first use and reuses them on subsequent starts for the same instance. When the agent container starts, the instance's disks are mounted at their declared paths, and the agent reads and writes state as a regular filesystem.

When a workload stops (idle timeout, crash, or explicit stop), those PVCs survive. On the next start for the same instance, the same PVCs are mounted and the agent resumes from its persisted state. Deletion of the instance (see [Agent Instances — Lifecycle](../agent-instances.md#lifecycle)) is what eventually releases them. An agent whose environment declares no persistent volume goes through the same sequence with nothing retained: every start begins from an empty filesystem, and durable conversation history remains in [Threads](../threads.md) regardless.

## Isolation

Agent state is one of three distinct data concerns in the platform, each with its own storage and lifecycle:

| Concern | What it stores | Storage | Lifetime |
|---------|---------------|---------|----------|
| **Chat / Threads** | User messages and agent responses | PostgreSQL ([Threads](../threads.md)) | Long-lived. The conversation record |
| **Agent state** | Internal working memory — conversation context, summaries, tool state, any agent-specific data | Disk, persistent or ephemeral per the environment | Lives as long as the disk under it |
| **Tracing** | Full LLM call context (complete request bodies sent to the model) for observability and debugging | PostgreSQL ([Tracing](../tracing.md)) | Shorter retention due to data volume |

These concerns are cleanly separated:

- **Threads** is the shared conversation record visible to all participants. Already stored in PostgreSQL. No changes.
- **Agent state** is private to the agent. No external service reads or writes it. The agent manages its own state without platform involvement.
- **Tracing** captures the full LLM context for each request as observability data. Due to the volume of data (complete message arrays per LLM call), tracing data has shorter retention than conversation records.
