# Agent State

## Overview

Agent state is managed entirely by each agent implementation and persisted locally on disk. The platform provides a persistent volume — the agent decides what to store and how to organize it. There is no platform-level state service, schema, or contract for agent state.

## Design Principles

- **Agent-owned.** Each agent implementation (our [`agn`](../agn-cli.md), wrapped Codex, wrapped Claude, custom agents) defines its own state format and storage layout. The platform does not interpret or validate agent state.
- **Disk-based.** State is written to a filesystem path backed by a persistent volume. No external service dependency for state persistence.
- **Instance-scoped.** State is keyed by [agent instance](../agent-instances.md), not by thread or by class. Multiple threads that route to the same instance share the same state; different instances of the same class have independent state.
- **Volume-scoped lifetime.** State lives as long as the persistent volume exists. The volume is bound to the instance; deleting the instance releases it (subject to volume TTL).

## How It Works

The [Agents Orchestrator](../agents-orchestrator.md) assembles the workload spec with persistent volumes defined as [Volume](../resource-definitions.md#volume) resources, keyed by `agent_instance_id` in the [Runners](../runners.md#volume-resource) records (distinct from the runner-local `instance_id`, which is the PVC name). The [Runner](../runner.md) creates PersistentVolumeClaims on first use and reuses them on subsequent starts for the same instance. When the agent container starts, the instance's persistent volume is mounted at the configured path, and the agent reads/writes state to it as a regular filesystem.

When a workload stops (idle timeout, crash, or explicit stop), the PVC survives. On the next start for the same instance, the same PVC is mounted and the agent resumes from its persisted state. Deletion of the instance (see [Agent Instances — Lifecycle](../agent-instances.md#lifecycle)) is what eventually releases the volume.

## Isolation

Agent state is one of three distinct data concerns in the platform, each with its own storage and lifecycle:

| Concern | What it stores | Storage | Lifetime |
|---------|---------------|---------|----------|
| **Chat / Threads** | User messages and agent responses | PostgreSQL ([Threads](../threads.md)) | Long-lived. The conversation record |
| **Agent state** | Internal working memory — conversation context, summaries, tool state, any agent-specific data | Disk (persistent volume) | Lives as long as the volume exists |
| **Tracing** | Full LLM call context (complete request bodies sent to the model) for observability and debugging | PostgreSQL ([Tracing](../tracing.md)) | Shorter retention due to data volume |

These concerns are cleanly separated:

- **Threads** is the shared conversation record visible to all participants. Already stored in PostgreSQL. No changes.
- **Agent state** is private to the agent. No external service reads or writes it. The agent manages its own state without platform involvement.
- **Tracing** captures the full LLM context for each request as observability data. Due to the volume of data (complete message arrays per LLM call), tracing data has shorter retention than conversation records.
