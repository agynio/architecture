# Agents Service

## Overview

The Agents service manages agent resources — the configuration entities that define agents and their dependencies.

This is a **control plane** service. It stores desired state; other services reconcile toward it.

Agents and Volumes are scoped to an [organization](organizations.md) (direct `organization_id`). Sub-resources (MCPs, Skills, Hooks, ENVs, InitScripts, Volume Attachments) inherit organization scope through their parent. See [Organizations — Resource Scoping](organizations.md#resource-scoping).

## API

Defined in `agynio/api` at `proto/agynio/api/agents/v1/agents.proto`. Exposed externally through the [Gateway](gateway.md) via ConnectRPC.

## Resources

| Resource | Description | CRUD |
|----------|-------------|------|
| **Agents** | Agent definitions: identity, model, image, init image, compute resources, behavioral configuration | ✓ |
| **Volumes** | Volume definitions: persistence, mount path, size | ✓ |
| **Volume Attachments** | Relationships between volumes and containers (agents, MCPs, hooks) | Create, Get, Delete, List |
| **MCPs** | MCP server definitions: image, command, compute resources. Belong to an agent | ✓ |
| **Skills** | Reusable prompt fragments: name, body. Belong to an agent | ✓ |
| **Hooks** | Event-driven functions: event, entrypoint, image, compute resources. Belong to an agent | ✓ |
| **ENVs** | Environment variables: name, plain value or secret reference. Belong to an agent, MCP, or hook | ✓ |
| **InitScripts** | Shell scripts for container initialization. Belong to an agent, MCP, or hook | ✓ |

All list endpoints use cursor-based pagination.

## Entity Model

All resources share a common `EntityMeta` base:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique identifier |
| `created_at` | timestamp | Creation time |
| `updated_at` | timestamp | Last modification time |

Resource-specific fields and ownership relationships are documented in [Resource Definitions](resource-definitions.md).
