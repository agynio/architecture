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
| **Agents** | Agent definitions: identity, model, image, init image, compute resources, behavioral configuration, [availability](#availability) | ✓ |
| **Volumes** | Volume definitions: persistence, mount path, size | ✓ |
| **Volume Attachments** | Relationships between volumes and containers (agents, MCPs, hooks) | Create, Get, Delete, List |
| **Image Pull Secret Attachments** | Relationships between [image pull secrets](providers.md#image-pull-secret) and containers (agents, MCPs, hooks). The image pull secret resource itself is managed by the [Secrets](secrets.md) service | Create, Get, Delete, List |
| **MCPs** | MCP server definitions: image, command, compute resources. Belong to an agent | ✓ |
| **Skills** | Reusable prompt fragments: name, body. Belong to an agent | ✓ |
| **Hooks** | Event-driven functions: event, entrypoint, image, compute resources. Belong to an agent | ✓ |
| **ENVs** | Environment variables: name, plain value or secret reference. Belong to an agent, MCP, or hook | ✓ |
| **InitScripts** | Shell scripts for container initialization. Belong to an agent, MCP, or hook | ✓ |

All list endpoints use cursor-based pagination.

### Egress Rules (managed elsewhere)

[Egress Rules](resource-definitions.md#egress-rule) — which control outbound HTTP/HTTPS from agent workloads — are **not** owned by the Agents service. They live in the dedicated [EgressRules service](egress-rules-service.md), attached to agents via that service's `EgressRuleAttachment` resource. The Agents service is unaware of rules.

## Availability

Every agent declares an `availability` value on its resource record (see [Resource Definitions — Agent](resource-definitions.md#agent)). The value gates who may initiate threads with the agent — that is, who may pass the agent as an initial participant on `CreateThread` or as the target of `AddParticipant`. Availability does **not** affect agent metadata visibility: `ListAgents` and the metadata view of `GetAgent` return every agent in the organization regardless of availability, so chat composers, group threads, and Console listings continue to display the agent's name and nickname. The check fires in the [Threads](threads.md#agent-availability-check) service at participation time.

| Value | Who can initiate threads |
|-------|--------------------------|
| `internal` | Any org member, plus any identity holding an [agent role](#roles) |
| `private` | Only identities holding an [agent role](#roles) (`owner`, `maintainer`, or `participant`) |

Apps that hold `participant:add` or `thread:create` on the organization are not exempt — an app adding a `private` agent must hold an agent role on it.

`availability` is a required field on `CreateAgent`; the API has no default. The Console prefills the create form with `internal`. Toggling availability via `UpdateAgent` does not retroactively affect existing thread participation — agents that are already thread participants stay until the thread is archived.

## Roles

Each agent has its own role list, separate from organization membership. A role is a direct relationship from an identity to a specific agent, expressed as an OpenFGA tuple on the [`agent` type](authz.md#agent). An identity holds **at most one** role per agent — assigning a new role replaces the existing one.

| Role | Capabilities |
|------|--------------|
| `owner` | Manage roles, change availability, delete the agent, edit and read configuration, initiate threads with the agent |
| `maintainer` | Edit and read configuration, initiate threads with the agent |
| `participant` | Initiate threads with the agent. No configuration access |

Role assignments are **not** stored in the Agents service database. They are OpenFGA tuples managed entirely through the [Authorization](authz.md) service. The Agents service exposes the [Agent Role API](#agent-role-api) as a thin domain wrapper — it does not maintain a parallel role table, has no PostgreSQL schema for roles, and is not the source of truth for role data. The same pattern is used by Organizations for [memberships](organizations.md#members-management) and by Apps for [installation permissions](apps.md#permissions-bridge).

Organization owners retain administrative access to every agent in their organization — they implicitly satisfy `owner`-level capabilities through the `owner from org` derivations on the `agent` type, regardless of any per-agent role. Agent role assignment is therefore most useful for granting non-owners scoped access to specific agents.

On `CreateAgent`, the calling identity is granted the `owner` role on the new agent automatically. Additional roles are managed through the [Agent Role API](#agent-role-api).

Role assignments are restricted to identities that are members of the agent's organization. Cross-organization role grants are rejected. ([Open question](#future-cross-organization-availability) on cross-org access.)

`agynd` reads its own configuration through the agent identity's existing `member` relation on the organization (see [agynd Startup Fetch](#agynd-startup-fetch)); the role model gates access by other identities and does not alter agent self-read.

## Agent Role API

The Agents service exposes role management RPCs as the external entry point — the [Authorization](authz.md) service is internal-only, so clients (Console, `agyn` CLI) reach role data through these RPCs. Each method is a thin wrapper that performs the domain check (target identity must be a member of the agent's organization), then issues `Read` / `Write` / `ListObjects` / `ListUsers` calls against the Authorization service. No per-role state is stored locally.

| Method | Description |
|--------|-------------|
| **SetAgentRole** | Assign or change the role of an identity on an agent. Atomically deletes any existing role tuple for the identity on this agent and writes the new one in a single Authorization `Write` call. Rejects identities that are not members of the agent's organization |
| **RemoveAgentRole** | Remove an identity's role on an agent. Issues a single Authorization `Write` with a delete |
| **ListAgentRoles** | List role assignments on an agent. Backed by Authorization `Read` filtered by `object = agent:<id>`. Returns `(identity_id, role)` pairs. Paginated |
| **ListMyAgentRoles** | List the caller's role assignments across every agent the caller holds a role on. Backed by Authorization `ListObjects(user=identity:<caller>, type=agent)` repeated per role and merged. Returns `(agent_id, role)` pairs. Self-only — does not accept an identity parameter |

`SetAgentRole`, `RemoveAgentRole`, and `ListAgentRoles` require `can_manage_roles` on the agent. `ListMyAgentRoles` is self-only and accepts only authenticated context. Authorization details: [Authorization — Agents Service](authz.md#agents-service).

## Internal API

| Method | Description |
|--------|-------------|
| `ResolveAgentIdentity` | Resolves an OpenZiti platform `identity_id` to the corresponding `agent_id` and `organization_id` |

This method derives agent and organization attribution from an authenticated OpenZiti connection identity. It is internal only — not exposed through the [Gateway](gateway.md).

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

Agent access is split into two layers: organization-level gates the creation and listing of agents and the visibility of agent metadata; per-agent role gates configuration access and thread initiation. The `agent` OpenFGA type is defined in [Authorization — Agent type](authz.md#agent).

| Operation | Check |
|-----------|-------|
| `CreateAgent`, `CreateVolume` | `owner` on `organization:<org_id>` |
| `ListAgents`, `GetAgent` (metadata) | `member` on `organization:<org_id>` — returns metadata fields only (`id`, `name`, `nickname`, `role`, `description`, `availability`, `created_at`, `updated_at`). Configuration fields are omitted unless the caller also satisfies `can_read_config` on the agent |
| `GetAgent` (configuration), `ListMCPs`, `ListSkills`, `ListHooks`, `ListENVs`, `ListInitScripts`, `ListVolumeAttachments`, `ListImagePullSecretAttachments` and their `Get` counterparts | `can_read_config` on `agent:<agent_id>` (i.e., agent `owner` or `maintainer`, or org `owner`) |
| `UpdateAgent` (configuration fields), Create/Update/Delete on any agent sub-resource (MCP, Skill, Hook, ENV, InitScript, Volume Attachment, Image Pull Secret Attachment) | `can_edit_config` on `agent:<agent_id>` |
| `UpdateAgent` (`availability` field), `DeleteAgent` | `can_delete` on `agent:<agent_id>` (agent `owner` or org `owner`) |
| `SetAgentRole`, `RemoveAgentRole`, `ListAgentRoles` | `can_manage_roles` on `agent:<agent_id>` (agent `owner` or org `owner`) |
| `ListMyAgentRoles` | Self only — returns the caller's own role assignments |
| Update, Delete on Volume | `owner` on `organization:<volume.org_id>` |
| Get, List on Volume (via Gateway) | `member` on `organization:<volume.org_id>` |

`SetAgentRole` rejects identities that are not members of the agent's organization. The check is performed against the `member` relation on the agent's org before the role tuple is written.

Agent workload identities (`identity_type == "agent"`) satisfy `member` on their organization and may call read APIs needed for self-configuration, including `ListENVs`. `ListENVs` never returns resolved secret values — secret-backed ENVs return only the `secret_id` reference. The role model gates access by other identities and does not alter agent self-read.

`ResolveAgentIdentity` is internal only — not exposed through the [Gateway](gateway.md) — and has no OpenFGA check.

The [Agents Orchestrator](agents-orchestrator.md) calls all Get/List methods over Istio for [workload spec assembly](agents-orchestrator.md#workload-spec-assembly). These internal reads are not exposed through the [Gateway](gateway.md), bypass all `member` and per-agent checks, and are gated by [Istio `AuthorizationPolicy`](authz.md#internal-rpc-authorization) restricted to the Orchestrator's ServiceAccount.

See [Authorization — Agents Service](authz.md#agents-service) for the full reference.

## Tuple Lifecycle

The Agents service is the writer of OpenFGA tuples on the `agent` type. All writes and deletes are issued through the [Authorization](authz.md) service; the Agents service does not store any of these tuples locally.

| Event | Tuples written | Tuples deleted |
|-------|----------------|----------------|
| `CreateAgent` | `organization:<org_id>, org, agent:<id>`; `identity:<creator>, owner, agent:<id>`; if `availability=internal`: `organization:<org_id>, internal_access, agent:<id>` | — |
| `UpdateAgent` toggles availability `private → internal` | `organization:<org_id>, internal_access, agent:<id>` | — |
| `UpdateAgent` toggles availability `internal → private` | — | `organization:<org_id>, internal_access, agent:<id>` |
| `SetAgentRole(identity, role)` (new identity) | `identity:<id>, <role>, agent:<agent_id>` | — |
| `SetAgentRole(identity, role)` (identity already holds a different role) | `identity:<id>, <new_role>, agent:<agent_id>` | `identity:<id>, <old_role>, agent:<agent_id>` |
| `RemoveAgentRole(identity)` | — | `identity:<id>, <role>, agent:<agent_id>` |
| `DeleteAgent` | — | All tuples on `agent:<id>` (`org`, `internal_access`, every `owner`/`maintainer`/`participant`) |

Tuple writes and deletes are issued in the same `Write` call as the underlying DB mutation when atomicity is required (see [Authorization — Relationship Writes](authz.md#relationship-writes)).

## Future: Cross-organization Availability

A future `public` availability value may permit identities outside the agent's organization to be assigned roles. The current model rejects such assignments. See [Open Questions](../open-questions.md) for the deferred design.

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
