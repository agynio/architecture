# Agents Service

## Overview

The Agents service manages agent resources — the configuration entities that define agents and their dependencies.

This is a **control plane** service. It stores desired state; other services reconcile toward it.

Agents and Volumes are scoped to an [organization](organizations.md) (direct `organization_id`). Sub-resources (MCPs, Skills, Hooks, ENVs, InitScripts, Volume Attachments, Image Pull Secret Attachments) inherit organization scope through their parent. See [Organizations — Resource Scoping](organizations.md#resource-scoping).

Agent nicknames are part of the agent resource. The Agents service stores the nickname and registers it with the [Identity](identity.md) service on create and update. Nickname uniqueness within the organization is enforced by the Identity service.

## API

Defined in `agynio/api` at `proto/agynio/api/agents/v1/agents.proto`. Exposed externally through the [Gateway](gateway.md) via ConnectRPC.

## Resources

| Resource | Description | CRUD |
|----------|-------------|------|
| **Agents** | Agent definitions: identity, model, image, init image, compute resources, behavioral configuration | ✓ |
| **Volumes** | Volume definitions: persistence, mount path, size | ✓ |
| **Volume Attachments** | Relationships between volumes and containers (agents, MCPs, hooks) | Create, Get, Delete, List |
| **Image Pull Secret Attachments** | Relationships between [image pull secrets](providers.md#image-pull-secret) and containers (agents, MCPs, hooks). The image pull secret resource itself is managed by the [Secrets](secrets.md) service | Create, Get, Delete, List |
| **MCPs** | MCP server definitions: image, command, compute resources. Belong to an agent | ✓ |
| **Skills** | Reusable prompt fragments: name, body. Belong to an agent | ✓ |
| **Hooks** | Event-driven functions: event, entrypoint, image, compute resources. Belong to an agent | ✓ |
| **ENVs** | Environment variables: name, plain value or secret reference. Belong to an agent, MCP, or hook | ✓ |
| **InitScripts** | Shell scripts for container initialization. Belong to an agent, MCP, or hook | ✓ |

All list endpoints use cursor-based pagination.

## Internal API

| Method | Description |
|--------|-------------|
| `ResolveAgentIdentity` | Resolves an OpenZiti platform `identity_id` to the corresponding `agent_id` and `organization_id` |

This method is used by the [Tracing](tracing.md) service to derive agent and organization attribution from the authenticated OpenZiti connection identity. It is not exposed through the [Gateway](gateway.md).

**Request:**

| Field | Type | Description |
|-------|------|-------------|
| `identity_id` | string (UUID) | Platform identity UUID (from OpenZiti) |

**Response:**

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | string (UUID) | Agent resource UUID |
| `organization_id` | string (UUID) | Organization the agent belongs to |

Returns `NOT_FOUND` if the identity does not correspond to an agent.

## agynd Startup Fetch

On startup, [`agynd`](agynd-cli.md) fetches the agent's full configuration from the Agents service via the Gateway. It authenticates using its own agent OpenZiti identity — the pod's Ziti sidecar handles mTLS transparently. `agynd` reads its own `agent_id` from the `AGENT_ID` environment variable (injected by the Orchestrator) and passes it explicitly in each API call.

The following resources are fetched before the agent CLI is spawned:

| Resource | Method | Purpose |
|----------|--------|---------|
| Agent | `GetAgent` | Base configuration: model, image, behavioral config |
| ENVs | `ListENVs(agent_id)` | Environment variables injected into the agent subprocess |
| Skills | `ListSkills(agent_id)` | Prompt fragments placed on the filesystem for the agent CLI |
| MCPs | `ListMCPs(agent_id)` | MCP server definitions — used to configure agent CLI MCP endpoints |
| InitScripts | `ListInitScripts(agent_id)` | Shell scripts executed before the agent CLI is spawned |

Each sub-resource list is fetched as a separate call. Secret-backed ENVs are resolved via the [Secrets](secrets.md) service after the ENV list is retrieved.

## Entity Model

All resources share a common `EntityMeta` base:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique identifier |
| `created_at` | timestamp | Creation time |
| `updated_at` | timestamp | Last modification time |

Resource-specific fields and ownership relationships are documented in [Resource Definitions](resource-definitions.md).
