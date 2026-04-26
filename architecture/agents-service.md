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

## Notifications

The Agents service publishes events to the [Notifications](notifications.md) service on the `agent:{id}` room so subscribers can react to configuration changes without polling.

| Event | Emitted when |
|-------|--------------|
| `agent.updated` | The agent resource is created, updated, or deleted, *or* any of its sub-resources (MCP, Skill, Hook, ENV, InitScript, Volume, Volume Attachment, Image Pull Secret Attachment) is created, updated, or deleted |

**Transitive `updated_at` propagation.** A successful write to a sub-resource bumps the owning agent's `updated_at` in the same transaction. Consumers that compare timestamps — for example, the [Agents Orchestrator's Start Decision](agents-orchestrator.md#start-decision), which uses `Agent.updated_at > failed_workload.removed_at` to decide whether a configuration change warrants a retry — therefore only need to read the agent record. No traversal of sub-resources is required.

Room subscription authorization is documented in [Notifications — Authorization](notifications.md#authorization): `member` on the agent's organization.

## Authorization

All agent resources are org-scoped. Access is determined by the caller's relation on the organization:

| Operation | Required relation on org |
|-----------|--------------------------|
| Create, Update, Delete (any resource) | `owner` |
| Get, List (any resource, via Gateway) | `member` |

Agent workload identities (`identity_type == "agent"`) satisfy `member` and may call all read APIs including `ListENVs`. `ListENVs` never returns resolved secret values — secret-backed ENVs return only the `secret_id` reference.

`ResolveAgentIdentity` is internal only — called by [Tracing](tracing.md) over Istio — and has no OpenFGA check.

The [Agents Orchestrator](agents-orchestrator.md) calls all Get/List methods over Istio for [workload spec assembly](agents-orchestrator.md#workload-spec-assembly). These internal reads are not exposed through the [Gateway](gateway.md), bypass the `member` check, and are gated by [Istio `AuthorizationPolicy`](authz.md#internal-rpc-authorization) restricted to the Orchestrator's ServiceAccount.

See [Authorization — Agents Service](authz.md#agents-service) for the full reference.

## agynd Startup Fetch

On startup, [`agynd`](agynd-cli.md) fetches agent configuration from the Agents service via the Gateway. It authenticates using its own agent OpenZiti identity — the pod's Ziti sidecar handles mTLS transparently. `agynd` reads its own `agent_id` from the `AGENT_ID` environment variable and passes it explicitly in each API call.

The following resources are fetched before the agent CLI is spawned:

| Resource | Method | Purpose |
|----------|--------|---------|
| Agent | `GetAgent` | Base configuration: model, image, behavioral config |
| Skills | `ListSkills(agent_id)` | Prompt fragments placed on the filesystem for the agent CLI |
| MCPs | `ListMCPs(agent_id)` | MCP server definitions — used to configure agent CLI MCP endpoints |
| InitScripts | `ListInitScripts(agent_id)` | Shell scripts executed before the agent CLI is spawned |

Environment variables are **not** fetched via API. The Orchestrator injects all ENV values (both plain-text and resolved secret values) directly into the container at assembly time. `agynd` reads them from the process environment.

## Entity Model

All resources share a common `EntityMeta` base:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique identifier |
| `created_at` | timestamp | Creation time |
| `updated_at` | timestamp | Last modification time |

Resource-specific fields and ownership relationships are documented in [Resource Definitions](resource-definitions.md).
