# Agents Service

## Overview

The Agents service manages agent resources — the configuration entities that define agents and their dependencies.

This is a **control plane** service. It stores desired state; other services reconcile toward it.

Agents, Environments, and Sandboxes are scoped to an [organization](organizations.md) (direct `organization_id`). Sub-resources (Volumes, MCPs, Skills, ENVs, InitScripts) inherit organization scope through their parent. See [Organizations — Resource Scoping](organizations.md#resource-scoping).

Agent nicknames are part of the agent resource. The Agents service stores the nickname and registers it with the [Identity](identity.md) service on create and update. Nickname uniqueness within the organization is enforced by the Identity service.

## API

Defined in `agynio/api` at `proto/agynio/api/agents/v1/agents.proto`. Exposed externally through the [Gateway](gateway.md) via ConnectRPC.

## Resources

| Resource | Description | CRUD |
|----------|-------------|------|
| **Agents** | Agent class definitions: identity, model, [environment](resource-definitions.md#environment) reference, behavioral configuration, [availability](#availability). See [Agents vs. Agent Instances](#agents-vs-agent-instances) | ✓ |
| **Environments** | Runtime definitions: runner reference + [flavor](resource-definitions.md#flavor) name + a workspace [image](resource-definitions.md#image) and optional agent runtime image, each with a tag. Referenced by agents and sandboxes; owner of the volumes, MCPs, init scripts, and ENVs a workload in it carries, and attachment target for egress rules. Create/update validates that the runner is visible to the organization and that each image reference resolves to a visible image of the required type with an existing tag; the flavor name is not validated — it is late-bound against the runner's [reported catalog](runners.md#runner-catalog) at workload start. Delete is rejected while any agent or sandbox references the environment. See [Flavors and Environments](../product/environments/environments.md) | ✓ |
| **Sandboxes** | On-demand workloads started by users: name, environment reference, owner, status, idle timeout, TTL, and the identities the owner has shared it with. `CreateSandbox` accepts an `idle_timeout`, validated against the organization's `sandbox_max_idle_timeout` and falling back to its `sandbox_default_idle_timeout`; both the resolved value and the TTL are stored on the sandbox. Reconciled by the [Agents Orchestrator](agents-orchestrator.md). See [Sandboxes](../product/sandboxes/sandboxes.md) | Create, Get, List, Stop, Delete, Share, Unshare, ListShares |
| **Agent Instances** | Instantiations of a class with their own state and [inbox](agent-instances.md#inbox). See [Agent Instances](agent-instances.md) | Create, Get, List, Pause, Resume, Delete |
| **Inbox Items** | Sub-resource of an instance. Written by Threads (fan-out from `SendMessage`) or by apps (direct writes). Read and acked by `agynd`. See [Agent Instances — Inbox](agent-instances.md#inbox) | Write (apps), List (self), Ack (self) |
| **Volumes** | Mount declarations: name, mount path, persistence, size, storage class, TTL. Belong to an environment or an MCP. A definition, not a disk — see [Resource Definitions — Volume](resource-definitions.md#volume) | ✓ |
| **MCPs** | MCP server definitions: [image](resource-definitions.md#image) reference + tag, command, compute resources, shared volume names. Belong to an environment or an agent | ✓ |
| **Skills** | Reusable prompt fragments: name, body. Belong to an agent | ✓ |
| **ENVs** | Environment variables: name, plain value or secret reference. Belong to an agent, MCP, or environment | ✓ |
| **InitScripts** | Shell scripts for container initialization. Belong to an environment, agent, or MCP | ✓ |

All list endpoints use cursor-based pagination.

## Agents vs. Agent Instances

Agents describe **what** an agent is (environment, model, prompt, tools). Agent Instances are **specific running (or eligible-to-run) copies** — each with its own state volume, its own inbox, and its own identity. Threads, workloads, and inbox items always reference an instance; the class is only referenced transiently, at participant-add time, before being resolved to a fresh instance (see [Threads — Class-on-Add Rewrite](threads.md#class-on-add-rewrite)).

Instances read **live class configuration** — there is no per-instance config snapshot. Consequently, `DeleteAgent` is rejected while the class has non-terminated instances; callers terminate the instances first (or let idle GC and retention run their course). See [Agent Instances — Configuration](agent-instances.md#configuration).

The full entity model, inbox schema, routing rules, and lifecycle live in [Agent Instances](agent-instances.md). This service owns the CRUD surface for both resources.

## Agent Instance API

| Method | Description |
|--------|-------------|
| **CreateInstance** | Create an instance of a class. Takes an optional `context` describing the circumstances of the creation — see [Creation Context](#creation-context). Called by Threads during class-on-add rewrite, which supplies `context.thread_id`; this service applies the class's [`default_thread`](resource-definitions.md#agent) policy to it, storing it as [`default_thread_id`](agent-instances.md#default-thread) under `origin` and ignoring it under `none`. Also exposed to callers directly (e.g., `agyn agents instantiate`), where an explicitly supplied `default_thread_id` is honored regardless of the policy. Enforces the [Agent Availability Check](threads.md#agent-availability-check) against the class. Returns a conflict error when the requested `label` is already taken by a non-terminated instance of the same class |
| **GetInstance** | Fetch instance record (id, agent_id, state, pause_reason, label, `default_thread_id`, timestamps) |
| **SetInstanceDefaultThread** | Set or clear the instance's [`default_thread_id`](agent-instances.md#default-thread). Requires `can_manage` on the instance. Creation sets this field; this is the only way it changes afterwards — joining further threads does not move it |
| **ListInstances** | List instances in an organization with server-side sort/filter/pagination. Filters include `agent_id`, `state_in`, and `has_unacked` (true when the instance has unacked inbox items). The Orchestrator uses `state=active, has_unacked=true` for its desired-state query |
| **PauseInstance** | Transition `active → paused` with a `pause_reason`. Called by the Orchestrator on unrecoverable instance failures (start failures exhausted, volume lost, runner deprovisioned), by the [idle GC](#idle-gc), or manually by an authorized caller. Inbox continues to accept writes |
| **ResumeInstance** | Transition `paused → active` and clear `pause_reason`. Pending inbox items are picked up on the Orchestrator's next reconciliation tick |
| **DeleteInstance** | Transition to `terminated`. Inbox rejects further writes. State volume TTL and cleanup follow the standard [Runners volume reconciliation](agents-orchestrator.md#volume-reconciliation) |

#### Creation Context

`CreateInstance` takes an optional `context` object describing **the circumstances the instance is being created in**, as plain facts. Callers report what they know; this service decides what any of it means, according to the class definition. Nothing in `context` is a request.

| Field | Type | Description |
|-------|------|-------------|
| `thread_id` | string (UUID) | The thread whose class-on-add rewrite triggered the creation. Supplied by [Threads](threads.md#class-on-add-rewrite). Interpreted per the class's [`default_thread`](resource-definitions.md#agent) policy |

The object is the extension point for anything later creation paths need to report — a source app installation, an initiating identity, a scheduling origin. New circumstances arrive as new `context` fields interpreted by new class policy, without widening the top-level request or teaching callers what the fields are for.

### Inbox API

| Method | Description |
|--------|-------------|
| **WriteInboxItem** | Write an item directly to an instance's inbox with `source_kind=direct`. Used by apps to address an instance without joining a thread. Requires `can_write_inbox` on the instance — granted directly or via the app-level [`inbox:write`](apps.md#permissions) permission |
| **GetUnackedInboxItems** | List unacked items for an instance. Self-only — the caller must be the instance itself. Used by `agynd` |
| **AckInboxItems** | Acknowledge processed items. Self-only |
| **GetUnackedInboxCount** | Count-only complement of `GetUnackedInboxItems`. Accepts an optional `thread_id` filter — used by [Chat](chat.md) to derive per-thread pending state, and by the Orchestrator's desired-state query without the filter |

Fan-out from `Threads.SendMessage` uses an **internal-only** RPC (`FanoutInboxItem`) that bypasses the app permission check on `WriteInboxItem` — Threads is trusted to enforce thread participation before calling it. See [Authorization — Agents Service](authz.md#agents-service) (to be updated).

### Idle GC

Instances transition `active → paused` when idle for the class's `instance_idle_ttl` (default `30d`) without new inbox items. The Agents service runs a background loop (default 60s) that scans `active` instances and pauses those past TTL. This is instance-level idle detection, separate from the workload-level idle timeout owned by the [Agents Orchestrator](agents-orchestrator.md#idle-timeout).

### Egress Rules (managed elsewhere)

[Egress Rules](resource-definitions.md#egress-rule) — which control outbound HTTP/HTTPS from workloads — are **not** owned by the Agents service. They live in the dedicated [EgressRules service](egress-rules-service.md), attached to agents or environments via that service's `EgressRuleAttachment` resource. The Agents service is unaware of rules.

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

## Environment Roles

Environments carry the same two-layer model as agents, for the same reason: any org member may author one (`can_create_environment`), and the creator becomes its `owner`. What differs is the third role.

| Role | Capabilities |
|------|--------------|
| `owner` | Manage roles, change availability, delete the environment, edit and read configuration, run workloads in it |
| `maintainer` | Edit and read configuration, run workloads in it |
| `user` | Run workloads in it — start a sandbox, or point an agent at it. No configuration access |

`user` is the environment analogue of an agent's `participant`, and it is the role that matters most here. An environment's volumes, secret-backed ENVs, and credential-injecting egress rules are all reachable from a shell in any [sandbox](../product/sandboxes/sandboxes.md) started against it, so *using* an environment is a grant, not a side effect of being able to see it. `availability` sets the default reach: `internal` extends `can_use` to every org member, `private` limits it to role holders.

| Value | Who can run workloads in the environment |
|-------|------------------------------------------|
| `internal` | Any org member, plus any identity holding an environment role |
| `private` | Only identities holding an environment role (`owner`, `maintainer`, or `user`) |

Availability does not affect metadata visibility: `ListEnvironments` and the metadata view of `GetEnvironment` return every environment in the organization, so pickers stay populated — what a non-holder cannot do is read the configuration or start anything in it.

Role storage follows the agent pattern exactly: OpenFGA tuples on the [`environment` type](authz.md#environment), no parallel table, exposed through **SetEnvironmentRole**, **RemoveEnvironmentRole**, and **ListEnvironmentRoles** with the same semantics as their agent counterparts. Organization owners retain administrative access to every environment through the `owner from org` derivations.

### Environment Metadata

Environment metadata reads — `ListEnvironments` and the metadata view of `GetEnvironment` — carry a computed **`can_use`** boolean resolved for the calling identity.

Metadata visibility and the right to run are deliberately different questions, and the answer to the second is not derivable from the response without it. `availability=internal` implies `can_use` for any member, but a `private` environment's answer depends on a role tuple the caller cannot read. A client filling an environment picker would therefore have to offer every `private` environment and let `CreateSandbox` reject the choice afterwards — which teaches the wrong thing about who may do what, at the worst moment.

`can_use` is computed per request from the same check the write paths enforce; it is not stored, and it grants nothing. It exists so a picker can offer what the caller can actually run in.

## Sandbox Sharing

A [sandbox](../product/sandboxes/sandboxes.md) owner may grant other identities in the organization access to that one sandbox.

| Method | Description |
|--------|-------------|
| **ShareSandbox** | Adds a principal — an identity or a [group](groups-service.md) — as a `collaborator`. The target must satisfy `member` on the sandbox's organization, checked before the tuple is written, as `SetAgentRole` does. Idempotent |
| **UnshareSandbox** | Removes a principal's `collaborator` tuple |
| **ListSandboxShares** | Lists the sandbox's collaborators |

All three require `can_share` — the owner. Organization owners can delete a sandbox they do not own but cannot share it, matching their inability to attach to it.

A collaborator holds `can_read`, `can_connect`, and `can_stop`, and nothing else: they can list the sandbox, open sessions on it through the [Terminal Proxy](terminal-proxy.md), start it through `EnsureSandboxRunning`, and stop it. They cannot delete it and cannot share it further. The relation definitions are in [Authorization — sandbox](authz.md#sandbox).

**Sharing a sandbox does not share its environment.** `can_use` on an environment governs creating sandboxes in it; `collaborator` governs entering one that already exists. A collaborator cannot start a second sandbox in the environment, and the share writes no environment tuple. What the share does hand over is everything reachable from a shell — the environment's secret-backed ENVs, its credential-injecting egress rules, and its volumes — which is why the product surface names the consequence at the point of sharing.

Metering is unchanged: `FLAVOR_SECONDS` records stay attributed to the organization and labeled with the sandbox and its owner regardless of who started the workload.

Shares live on the sandbox, so they do not survive it. Deleting a sandbox and creating another starts with an empty share list.

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
| `agent_instance_id` | string (UUID) | Agent instance UUID (equals the input `identity_id`) |
| `agent_id` | string (UUID) | Agent class UUID, resolved through the instance |
| `organization_id` | string (UUID) | Organization the agent belongs to |

Returns `NOT_FOUND` if the identity does not correspond to an agent instance.

## Notifications

The Agents service publishes events to the [Notifications](notifications.md) service so subscribers can react to configuration changes without polling.

| Event | Room | Emitted when |
|-------|------|--------------|
| `agent.updated` | `agent:{class_id}` | The agent class is created, updated, or deleted, *or* any of its sub-resources (MCP, Skill, ENV, InitScript) is created, updated, or deleted |
| `environment.updated` | `environment:{environment_id}`, `environments` | The environment is created, updated, or deleted, *or* any of its sub-resources (Volume, MCP, InitScript, ENV) is created, updated, or deleted. Every agent and sandbox running the environment is affected, which is why the event is published once per environment rather than fanned out to each referencing agent. The flat `environments` room carries the same event for an infrastructure consumer that cannot enumerate the environments it serves — the [LLM Proxy](llm-proxy.md#guardrails) caches an environment's model allowlist on a connection, and a subscription's rooms are fixed when its stream opens |
| `message.created` | `instance_inbox:{instance_id}` | A new [inbox item](agent-instances.md#inbox) is written for an instance (via `FanoutInboxItem` from Threads or `WriteInboxItem` from an app) |
| `instance.updated` | `agent:{class_id}` | Instance state transitions (`active`, `paused`, `terminated`) or metadata changes on any instance of the class |
| `sandbox.updated` | `sandbox_owner:{owner_id}`, `sandbox:{sandbox_id}`, and `sandbox_org:{organization_id}` | Sandbox lifecycle or bookkeeping changes: create, status transition, stop, delete/terminate, `last_session_at` update, share list change, or workload association update |

**Transitive `updated_at` propagation.** A successful write to a sub-resource bumps its owner's `updated_at` in the same transaction — an agent sub-resource bumps the agent, an environment sub-resource bumps the environment, and an MCP sub-resource bumps the MCP's own owner. Consumers that compare timestamps — for example, the [Agents Orchestrator's Start Decision](agents-orchestrator.md#start-decision), which compares the later of the agent's and its environment's `updated_at` against `failed_workload.removed_at` to decide whether a configuration change warrants a retry — therefore only need to read the two parent records. No traversal of sub-resources is required.

Room subscription authorization is documented in [Notifications — Authorization](notifications.md#authorization). Agent rooms require `member` on the agent's organization. Sandbox owner rooms require the caller identity to match `{owner_id}`; sandbox org rooms require `can_list_sandboxes` on `organization:{organization_id}` so organization owners can maintain list-all views.

**Three rooms, because no one of them reaches everyone entitled to the event.** The owner room is identity-keyed and carries a member's whole list in one subscription, but cannot reach a [collaborator](authz.md#sandbox) — the sandbox is not theirs. The org room is scoped to `can_list_sandboxes`, which collaborators do not hold either. `sandbox:{sandbox_id}` closes the gap: one room per sandbox, gated by `can_read`, subscribed per shared sandbox by clients that display one.

`sandbox.updated` payloads include the sandbox ID, organization ID, owner ID, name, environment ID, status, idle timeout, TTL, `last_session_at`, and current workload ID when one exists. Terminated sandboxes are still emitted to every room so default lists can remove the row while `--terminated` views retain audit visibility.

## Authorization

Agent and environment access is split into two layers: organization-level gates creation, listing, and metadata visibility; the per-resource role gates configuration access, thread initiation (agents), and the right to run workloads (environments). The `agent` and `environment` OpenFGA types are defined in [Authorization](authz.md#agent).

| Operation | Check |
|-----------|-------|
| `CreateAgent` | `owner` on `organization:<org_id>` |
| `ListAgents`, `GetAgent` (metadata) | `member` on `organization:<org_id>` — returns metadata fields only (`id`, `name`, `nickname`, `role`, `description`, `availability`, `created_at`, `updated_at`). Configuration fields are omitted unless the caller also satisfies `can_read_config` on the agent |
| `GetAgent` (configuration), `ListMCPs`, `ListSkills`, `ListENVs`, `ListInitScripts` and their `Get` counterparts, scoped to an agent | `can_read_config` on `agent:<agent_id>` (i.e., agent `owner` or `maintainer`, or org `owner`) |
| `UpdateAgent` (configuration fields), Create/Update/Delete on any agent sub-resource (MCP, Skill, ENV, InitScript) | `can_edit_config` on `agent:<agent_id>` |
| `UpdateAgent` (`availability` field), `DeleteAgent` | `can_delete` on `agent:<agent_id>` (agent `owner` or org `owner`) |
| `SetAgentRole`, `RemoveAgentRole`, `ListAgentRoles` | `can_manage_roles` on `agent:<agent_id>` (agent `owner` or org `owner`) |
| `ListMyAgentRoles` | Self only — returns the caller's own role assignments |
| `CreateAgent` / `UpdateAgent` with an `environment_id` | Additionally `can_use` on `environment:<environment_id>` — pointing an agent at an environment is using it |
| `CreateEnvironment` | `can_create_environment` on `organization:<org_id>` (computed from `member`); the creator is granted the `owner` role |
| `ListEnvironments`, `GetEnvironment` (metadata) | `member` on `organization:<org_id>` — returns metadata only (`id`, `name`, runner, flavor, images, `availability`, timestamps). Volumes, MCPs, init scripts, and ENVs are omitted unless the caller also satisfies `can_read_config` |
| `GetEnvironment` (configuration), and List/Get on any environment sub-resource (Volume, MCP, InitScript, ENV) | `can_read_config` on `environment:<environment_id>` |
| `UpdateEnvironment` (configuration fields), Create/Update/Delete on any environment sub-resource | `can_edit_config` on `environment:<environment_id>` |
| `UpdateEnvironment` (`availability` field), `DeleteEnvironment` | `can_delete` on `environment:<environment_id>` |
| `SetEnvironmentRole`, `RemoveEnvironmentRole`, `ListEnvironmentRoles` | `can_manage_roles` on `environment:<environment_id>` |
| `CreateSandbox` | `can_create_sandbox` on `organization:<org_id>` and `can_use` on `environment:<environment_id>`; owner is derived from authenticated context |
| `GetSandbox` | `can_read` on `sandbox:<id>` — the owner, an identity the owner shared it with, or an org owner |
| `ListSandboxes` (own) | `member` on `organization:<org_id>` and server filters to sandboxes the caller owns or collaborates on |
| `ListSandboxes` (`all=true`) | `can_list_sandboxes` on `organization:<org_id>` |
| `StopSandbox`, `DeleteSandbox` | `can_stop` / `can_delete` on `sandbox:<id>` |
| `EnsureSandboxRunning` | `can_connect` on `sandbox:<id>` |
| `ShareSandbox`, `UnshareSandbox`, `ListSandboxShares` | `can_share` on `sandbox:<id>` — the owner only |
| `UpdateSandboxLastSession` | Internal only (Terminal Proxy via Istio) |

`SetAgentRole` rejects identities that are not members of the agent's organization. The check is performed against the `member` relation on the agent's org before the role tuple is written.

Agent workload identities (`identity_type == "agent_instance"`) satisfy `member` on their organization — resolved through the instance's `org` relation (see [Authorization — agent_instance](authz.md#agent_instance)) — and may call read APIs needed for self-configuration, including `ListENVs` against their class (via the instance's `class` relation). `ListENVs` never returns resolved secret values — secret-backed ENVs return only the `secret_id` reference. The role model gates access by other identities and does not alter instance self-read.

`ResolveAgentIdentity` is internal only — not exposed through the [Gateway](gateway.md) — and has no OpenFGA check.

The [Agents Orchestrator](agents-orchestrator.md) calls all Get/List methods over Istio for [workload spec assembly](agents-orchestrator.md#workload-spec-assembly). These internal reads are not exposed through the [Gateway](gateway.md), bypass all `member` and per-agent checks, and are gated by [Istio `AuthorizationPolicy`](authz.md#internal-rpc-authorization) restricted to the Orchestrator's ServiceAccount.

See [Authorization — Agents Service](authz.md#agents-service) for the full reference.

## Tuple Lifecycle

The Agents service is the writer of OpenFGA tuples on the `agent` and `sandbox` types. All writes and deletes are issued through the [Authorization](authz.md) service; the Agents service does not store any of these tuples locally.

| Event | Tuples written | Tuples deleted |
|-------|----------------|----------------|
| `CreateAgent` | `organization:<org_id>, org, agent:<id>`; `identity:<creator>, owner, agent:<id>`; if `availability=internal`: `organization:<org_id>, internal_access, agent:<id>` | — |
| `UpdateAgent` toggles availability `private → internal` | `organization:<org_id>, internal_access, agent:<id>` | — |
| `UpdateAgent` toggles availability `internal → private` | — | `organization:<org_id>, internal_access, agent:<id>` |
| `SetAgentRole(identity, role)` (new identity) | `identity:<id>, <role>, agent:<agent_id>` | — |
| `SetAgentRole(identity, role)` (identity already holds a different role) | `identity:<id>, <new_role>, agent:<agent_id>` | `identity:<id>, <old_role>, agent:<agent_id>` |
| `RemoveAgentRole(identity)` | — | `identity:<id>, <role>, agent:<agent_id>` |
| `DeleteAgent` | — | All tuples on `agent:<id>` (`org`, `internal_access`, every `owner`/`maintainer`/`participant`) |
| `CreateSandbox` | `organization:<org_id>, org, sandbox:<id>`; `identity:<creator_id>, owner, sandbox:<id>` | — |
| `ShareSandbox(principal)` | `identity:<principal_id>, collaborator, sandbox:<id>`, or `group:<group_id>#member, collaborator, sandbox:<id>` for a group | — |
| `UnshareSandbox(principal)` | — | The principal's `collaborator` tuple on `sandbox:<id>` |
| Sandbox hard purge (retention policy) | — | All tuples on `sandbox:<id>` (`org`, `owner`, every `collaborator`). `DeleteSandbox` and TTL expiry only transition status to `terminated` — tuples survive the soft state so owners keep audit read access. See [Authorization — sandbox](authz.md#sandbox) |

Tuple writes and deletes are issued in the same `Write` call as the underlying DB mutation when atomicity is required (see [Authorization — Relationship Writes](authz.md#relationship-writes)).

## Future: Cross-organization Availability

A future `public` availability value may permit identities outside the agent's organization to be assigned roles. The current model rejects such assignments. See [Open Questions](../open-questions.md) for the deferred design.

## agynd Startup Fetch

On startup, [`agynd`](agynd-cli.md) fetches agent configuration from the Agents service via the Gateway. It authenticates using its own agent OpenZiti identity — the pod's Ziti sidecar handles mTLS transparently. `agynd` reads its own `agent_id` from the `AGENT_ID` environment variable and passes it explicitly in each API call.

The following resources are fetched before the agent CLI is spawned:

| Resource | Method | Purpose |
|----------|--------|---------|
| Agent | `GetAgent` | Base configuration: model, environment, behavioral config |
| Skills | `ListSkills(agent_id)` | Prompt fragments placed on the filesystem for the agent CLI |
| InitScripts | `ListInitScripts(environment_id)` then `ListInitScripts(agent_id)` | Shell scripts executed before the agent CLI is spawned, in that order |

A [sandbox](../product/sandboxes/sandboxes.md) workload has no agent, so its `agynd` fetches the environment-scoped list only.

MCP servers are **not** fetched via API either. The [Orchestrator](agents-orchestrator.md) resolves an agent's and its environment's MCPs when it assembles the workload — it is what builds the sidecars — and injects their names and ports as `AGENT_MCP_SERVERS`. `agynd` reads that and writes the agent CLI's endpoint list; it never asks the Agents service which MCPs exist, and holds no relationship to the environment that would let it.

Environment variables are **not** fetched via API. The Orchestrator injects all ENV values (both plain-text and resolved secret values) directly into the container at assembly time. `agynd` reads them from the process environment.

## Entity Model

All resources share a common `EntityMeta` base:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique identifier |
| `created_at` | timestamp | Creation time |
| `updated_at` | timestamp | Last modification time |

Resource-specific fields and ownership relationships are documented in [Resource Definitions](resource-definitions.md).
