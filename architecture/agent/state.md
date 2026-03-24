# Agent State

## Overview

Agent state is managed entirely by each agent implementation and persisted locally on disk. The platform provides a persistent volume — the agent decides what to store and how to organize it. There is no platform-level state service, schema, or contract for agent state.

## Design Principles

- **Agent-owned.** Each agent implementation (our [`agn`](../agn-cli.md), wrapped Codex, wrapped Claude, custom agents) defines its own state format and storage layout. The platform does not interpret or validate agent state.
- **Disk-based.** State is written to a filesystem path backed by a persistent volume. No external service dependency for state persistence.
- **Volume-scoped lifetime.** State lives as long as the persistent volume exists. The [Volume](../resource-definitions.md#volume) resource's lifecycle (creation, deletion) determines when state is created and lost.

## How It Works

The [Agents Orchestrator](../agents-orchestrator.md) assembles the workload spec with persistent volumes defined as [Volume](../resource-definitions.md#volume) resources. The [Runner](../runner.md) creates PersistentVolumeClaims on first use and reuses them on subsequent starts. When the agent container starts, the persistent volume is mounted at the configured path, and the agent reads/writes state to it as a regular filesystem.

When the agent is stopped (idle timeout, crash, or explicit stop), the PVC survives. On the next start, the same PVC is mounted, and the agent resumes from its persisted state.

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
