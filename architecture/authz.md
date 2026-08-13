# Authorization

## Overview

The platform uses [OpenFGA](https://openfga.dev) for fine-grained authorization. OpenFGA is a CNCF relationship-based access control (ReBAC) engine inspired by [Google Zanzibar](https://research.google/pubs/pub48190/). It evaluates access by traversing a graph of relationships between identities and resources.

A dedicated **Authorization** service sits in front of OpenFGA. All services call the Authorization service — no service communicates with OpenFGA directly.

## Authorization Service

The Authorization service is a thin gRPC proxy to OpenFGA. It centralizes the OpenFGA connection configuration (`store_id`, `model_id`, API endpoint), adds observability (metrics, tracing, structured logging), and provides a stable internal interface.

Services construct relationship tuples using their domain knowledge and send them through the Authorization service. The service does not interpret or transform tuples — it forwards them to OpenFGA with injected configuration.

### Interface

The Authorization service mirrors the OpenFGA runtime API:

| Method | Description |
|--------|-------------|
| **Check** | Can identity X perform relation Y on resource Z? Returns `allowed: bool` |
| **BatchCheck** | Multiple checks in a single call. Each check has a `correlation_id` for matching responses |
| **Write** | Write and/or delete relationship tuples atomically (up to 100 tuples per call) |
| **Read** | Read tuples matching a filter (user, relation, object — each optional). Paginated |
| **ListObjects** | What objects of type T does identity X have relation Y with? Returns list of object IDs |
| **ListUsers** | What identities have relation Y with object Z? Returns list of identity IDs |

Callers do not provide `store_id` or `authorization_model_id` — the Authorization service injects these from its own configuration.

### Classification

The Authorization service is a **data plane** service — it handles permission checks on the live request path.

## Authorization Model

The authorization model defines types, relations, and how permissions are computed. Written in OpenFGA's DSL and deployed via Terraform (see [Model Deployment](#model-deployment)).

### Identities

All platform identities (users, agents, runners, apps) are represented as a single `identity` type in OpenFGA. Services do not need to know the identity type when constructing tuples or performing checks — they use `identity:<identity_id>` uniformly. The `identity_type` distinction (from [Authentication](authn.md)) is orthogonal to the authorization model.

Any identity can hold any relationship that is modeled in OpenFGA. See [Identity](identity.md) for the identity registry.

### Types and Relations

#### cluster

Singleton object `cluster:global`. Holds platform-wide administrative permissions.

```
type cluster
  relations
    define admin: [identity]
```

#### organization

Organizations are the primary grouping unit. Resources (agents, models, secrets, threads) belong to an organization. Access to those resources derives from the caller's relation on the organization.

```
type organization
  relations
    define cluster: [cluster]
    define member: [identity]
    define owner: [identity]
    define can_invite: owner
    define can_manage_members: owner or admin from cluster
    define can_view_threads: owner or admin from cluster
    define can_view_workloads: owner or admin from cluster
    define can_view_volumes: owner or admin from cluster
    define can_add_member: admin from cluster
    define can_create_thread: member or thread_create
    define can_create_sandbox: member
    define can_create_environment: member
    define can_list_sandboxes: owner or admin from cluster
    define thread_create: [identity]
    define thread_write: [identity]
    define participant_add: [identity]
    define inbox_write: [identity]
```

`owner` implies `member`, `can_invite`, `can_manage_members`, `can_view_threads`, `can_view_workloads`, `can_view_volumes`, and `can_list_sandboxes`. `owner` does **not** imply `can_create_thread` or `can_create_sandbox` directly — instead those permissions are computed from `member`, and owners are also members, so owners can always create threads and sandboxes.

`thread_create`, `thread_write`, `participant_add`, and `inbox_write` are **app installation permissions** — direct relations written when an app is installed. See [App Installation Permissions](#app-installation-permissions). `inbox_write` is consumed by the [`agent_instance`](#agent_instance) type via `inbox_write from org`.

#### thread

Threads are independent objects in OpenFGA. Each thread holds its organization reference and its participant set as relationships.

```
type thread
  relations
    define org: [organization]
    define participant: [identity]
    define can_read: participant or can_view_threads from org
    define can_write: participant or thread_write from org
    define can_add_participant: participant or participant_add from org
```

When a thread is created, the Threads service writes:
- `organization:<org_id>, org, thread:<thread_id>`
- `identity:<id>, participant, thread:<thread_id>` for each initial participant

When `AddParticipant` is called, the Threads service writes:
- `identity:<id>, participant, thread:<thread_id>`

#### model

Models are org-scoped resources with explicit permissions for management and use.

```
type model
  relations
    define org: [organization]
    define can_manage: owner from org
    define can_use: member from org
```

When a model is created, the LLM service writes:
- `organization:<org_id>, org, model:<model_id>`

`can_manage` is computed from `owner` on the model's org. `can_use` is computed from `member` on the model's org. This makes it possible to grant or restrict model access independently in the future (e.g., writing a direct `identity:<id>, can_use, model:<model_id>` tuple) without changing the organization membership model.

#### agent

Agents are org-scoped resources with per-agent roles and an availability gate. The type defines three direct roles, an optional `internal_access` tuple that exposes the agent to all org members for thread initiation, and the derived permissions consumed by the [Agents Service](agents-service.md#authorization) and [Threads](threads.md#agent-availability-check). Direct roles accept individual identities and group members (`group#member`) so a single group grant covers every member transitively.

```
type agent
  relations
    define org: [organization]
    define owner: [identity, group#member]
    define maintainer: [identity, group#member]
    define participant: [identity, group#member]
    define internal_access: [organization]
    define can_initiate: owner or maintainer or participant or member from internal_access
    define can_read_config: owner or maintainer or owner from org
    define can_edit_config: owner or maintainer or owner from org
    define can_manage_roles: owner or owner from org
    define can_delete: owner or owner from org
```

Reading agent metadata (name, nickname, role label, description, availability) is gated by `member` on the agent's organization, the same check that backs `ListAgents` — it is not encoded as a relation on the `agent` type. Configuration fields and sub-resources are gated by `can_read_config`.

The `internal_access` relation accepts the `organization` type. When availability is `internal`, the Agents service writes `organization:<org_id>, internal_access, agent:<id>`. The derived `can_initiate` clause `member from internal_access` resolves any org member to `can_initiate`. When availability flips to `private`, the tuple is deleted and only explicit role holders remain.

`can_manage_roles` and `can_delete` include `owner from org` so organization owners retain administrative control over every agent in their org without having to be assigned a per-agent owner role. `can_read_config` and `can_edit_config` likewise include `owner from org`. The agent `owner` role is the elevated role for non-org-owners — typically the agent's creator or an identity an org owner has promoted via `SetAgentRole`.

The `participant` role on the `agent` type is distinct from the `participant` relation on the `thread` type — they share a name but live on different OpenFGA types and never conflict at evaluation time.

When an agent is created, the Agents service writes:
- `organization:<org_id>, org, agent:<id>`
- `identity:<creator_id>, owner, agent:<id>`
- If `availability=internal`: `organization:<org_id>, internal_access, agent:<id>`

Role mutations (`SetAgentRole`, `RemoveAgentRole`) and availability toggles update the corresponding tuples. `DeleteAgent` removes every tuple on `agent:<id>`. The full table is in [Tuple Lifecycle](#tuple-lifecycle) and [Agents Service — Tuple Lifecycle](agents-service.md#tuple-lifecycle).

Granting an agent role to a group writes the tuple `group:<group_id>#member, <role>, agent:<id>`. Any identity holding `member` on `group:<group_id>` then resolves the agent role transitively. See [Groups Service](groups-service.md) for the group type and member resolution.

#### agent_instance

[Agent instances](agent-instances.md) derive their authorization from their class. The type carries no roles of its own — lifecycle control follows the class's `can_delete`, thread-participation gating follows the class's `can_initiate`, and the only instance-local relation is direct inbox write access.

```
type agent_instance
  relations
    define class: [agent]
    define org: [organization]
    define can_initiate: can_initiate from class
    define can_manage: can_delete from class
    define can_write_inbox: [identity, group#member] or inbox_write from org
```

| Relation | Gates |
|----------|-------|
| `can_initiate` | Adding the instance to a thread (checked by [Threads](threads.md#agent-availability-check), resolved through the class) |
| `can_manage` | `PauseInstance`, `ResumeInstance`, `DeleteInstance` |
| `can_write_inbox` | `WriteInboxItem` — direct inbox writes. Held by identities granted directly (or via group), and by app identities whose installation declared the [`inbox:write` permission](apps.md#permissions) (org-level `inbox_write` tuple) |

Inbox reads and acks (`GetUnackedInboxItems`, `AckInboxItems`) are **self-only** — identity equality (`caller.identity_id == instance_id`), no OpenFGA check, same pattern as `TouchWorkload`.

When an instance is created, the Agents service writes:
- `agent:<class_id>, class, agent_instance:<id>`
- `organization:<org_id>, org, agent_instance:<id>`

`DeleteInstance` (terminal) removes every tuple on `agent_instance:<id>`.

#### environment

[Environments](resource-definitions.md#environment) are org-scoped resources authored by any member and owned by their creator. The type mirrors [`agent`](#agent), with one relation agents have no need for: `can_use`.

```
type environment
  relations
    define org: [organization]
    define owner: [identity, group#member]
    define maintainer: [identity, group#member]
    define user: [identity, group#member]
    define internal_access: [organization]
    define can_use: owner or maintainer or user or member from internal_access
    define can_read_config: owner or maintainer or owner from org
    define can_edit_config: owner or maintainer or owner from org
    define can_manage_roles: owner or owner from org
    define can_delete: owner or owner from org
```

**`can_use` separates running in an environment from editing one.** An environment carries secret-backed [ENVs](resource-definitions.md#env), egress rules that inject credentials, and the contents of its [volumes](resource-definitions.md#volume) — and anyone who can start a [sandbox](../product/sandboxes/sandboxes.md) in it gets an interactive shell sitting on all three. Being able to *use* an environment is therefore a grant in its own right, and it is checked on `CreateSandbox` and on any `CreateAgent`/`UpdateAgent` that points an agent at the environment. `can_edit_config`, which governs the volumes, MCPs, init scripts, and ENVs the environment declares, is a strictly separate question: a team may hand out `user` widely while keeping authorship to itself.

Reading environment *metadata* (name, runner, flavor, images, availability) is gated by `member` on the organization — the same split [`agent`](#agent) uses, and what lets any member see the environment picker without being able to read the ENVs behind it.

`availability` drives `internal_access` exactly as it does on agents: `internal` writes `organization:<org_id>, internal_access, environment:<id>`, resolving every org member to `can_use`; `private` deletes the tuple, leaving only explicit role holders. An environment holding credentials that only one team should reach is `private` with `user` granted to that team's group.

When an environment is created, the Agents service writes:
- `organization:<org_id>, org, environment:<id>`
- `identity:<creator_id>, owner, environment:<id>`
- If `availability=internal`: `organization:<org_id>, internal_access, environment:<id>`

Volumes, MCPs, init scripts, and ENVs owned by an environment carry no tuples of their own — they are gated by `can_read_config` / `can_edit_config` on the parent environment.

#### sandbox

[Sandboxes](../product/sandboxes/sandboxes.md) are org-scoped resources owned by the user who created them, and shareable by that owner with other identities in the organization. Organization owners can list, stop, and delete every sandbox in the organization, but attaching to one is never implied by org ownership: an org owner cannot connect to another user's sandbox unless they are that sandbox's `owner` or `collaborator`.

`can_connect` gates every session the [Terminal Proxy](terminal-proxy.md#session-kinds) issues a ticket for, not shells alone — a [workspace sync](sandbox-sync.md) session reaches the same filesystem a shell does and is authorized identically.

```
type sandbox
  relations
    define org: [organization]
    define owner: [identity]
    define collaborator: [identity, group#member]
    define can_read: owner or collaborator or owner from org
    define can_connect: owner or collaborator
    define can_stop: owner or collaborator or owner from org
    define can_delete: owner or owner from org
    define can_share: owner
```

**A share is a grant over one sandbox, not over its environment.** `can_use` on an [environment](#environment) governs whether an identity may *create* a sandbox there; `collaborator` governs who may enter a particular one. A collaborator needs only the second, and holding it grants nothing about the environment — they cannot start a second sandbox in it.

The consequence is deliberate: a shell reaches the environment's secret-backed ENVs, its credential-injecting egress rules, and its volumes, so an owner sharing a sandbox hands all three to someone who may hold no role on that environment. The product surface states this at the point of sharing — see [Sandboxes — Sharing](../product/sandboxes/sandboxes.md#sharing).

`can_stop` includes `collaborator` because `EnsureSandboxRunning` is gated by `can_connect`: a collaborator can already restart an idled-out sandbox, and being able to start what you work in without being able to stop it is not a boundary worth drawing. `can_delete` and `can_share` stay with the owner — a collaborator can neither destroy the sandbox nor pass the grant on.

When a sandbox is created, the Agents service writes:
- `organization:<org_id>, org, sandbox:<sandbox_id>`
- `identity:<creator_id>, owner, sandbox:<sandbox_id>`

`ShareSandbox` writes `identity:<target_id>, collaborator, sandbox:<sandbox_id>` (or `group:<group_id>#member` for a group principal); `UnshareSandbox` deletes it. Revocation takes effect at the next ticket issuance — a session already attached runs until it ends, as the Terminal Proxy validates tickets at issuance and not continuously.

Tuples are **retained through the `terminated` soft state** and removed only when the record is hard-purged (future retention policy). Terminated sandboxes stay readable for audit — the owner keeps `GetSandbox` on their own terminated sandboxes via the `owner` tuple. What terminated sandboxes can no longer *do* is gated by lifecycle status in the Agents service (`EnsureSandboxRunning`/`StopSandbox` reject `status=terminated`), not by tuple deletion: authorization answers who may act, status answers what is currently possible.

#### group

Groups are org-scoped collections of identities. The type defines membership, group-level admin, and computed view/edit permissions. See [Groups Service](groups-service.md) for the full lifecycle.

```
type group
  relations
    define org: [organization]
    define member: [identity]
    define admin: [identity]
    define can_view: member or member from org
    define can_edit: admin or owner from org
```

When a group is created, the Groups service writes:
- `group:<id>, org, organization:<org_id>`
- `identity:<creator_id>, admin, group:<id>` (optional — orgs may delegate group admin)

When a member is added, the Groups service writes:
- `identity:<member_id>, member, group:<id>`

Members are individual identities only in v1 — see [Groups Service — Future: Group Nesting](groups-service.md#future-group-nesting) for the planned `group#member` recursive extension.

Other types (agent, thread, organization, etc.) reference groups via `group#member` in their direct-role definitions. The `agent` type above is the first consumer; additional types will extend their roles similarly as group-based grants are needed.

### Organization Permissions

| Relation | Type | Capabilities |
|----------|------|-------------|
| **owner** | role (assignable) | Full access. Manage organization settings, membership, all resources. Delete organization |
| **member** | role (assignable) | Chat. View tracing. View resources (read-only). Create threads |
| **can_invite** | computed | Create pending memberships (invites) for the organization |
| **can_manage_members** | computed | Remove members, update member roles, list members |
| **can_add_member** | computed | Create active memberships directly (skip invite) |
| **can_view_threads** | computed | List and read all threads in the organization, regardless of participation. Held by owners and cluster admins |
| **can_view_workloads** | computed | List and read active workloads (and their containers and logs) in the organization. Held by owners and cluster admins |
| **can_view_volumes** | computed | List and read provisioned volumes in the organization. Held by owners and cluster admins |
| **can_create_thread** | computed | Create threads in the organization. Computed from `member` or `thread_create` |
| **can_create_sandbox** | computed | Create sandboxes in the organization. Computed from `member` |
| **can_create_environment** | computed | Create environments in the organization. Computed from `member` — environments are authored by whoever needs one, and the creator becomes their [`owner`](#environment) |
| **can_list_sandboxes** | computed | List all sandboxes in the organization. Held by owners and cluster admins |
| **thread_create** | app permission | Written for app installations that declare the `thread:create` permission |
| **thread_write** | app permission | Written for app installations that declare the `thread:write` permission |
| **participant_add** | app permission | Written for app installations that declare the `participant:add` permission |
| **inbox_write** | app permission | Written for app installations that declare the `inbox:write` permission. Flows into `can_write_inbox` on every [`agent_instance`](#agent_instance) in the org |

#### Computed relations

- `owner` implies `member`, `can_invite`, `can_manage_members`, `can_view_threads`, `can_view_workloads`, `can_view_volumes`, and `can_list_sandboxes`.
- `can_create_sandbox` and `can_create_environment` follow `member`; every active organization member can create a sandbox and author an environment.
- `can_add_member`, `can_manage_members`, `can_view_threads`, `can_view_workloads`, `can_view_volumes`, and `can_list_sandboxes` each include `admin from cluster` — any identity with the `admin` relation on `cluster:global` holds these permissions on every organization. Modeled as cross-type computed relations, not as explicit per-organization tuples. Cluster-admin listing does not extend to terminal attach — `can_connect` is held only by a sandbox's `owner` and the identities that owner has shared it with.
- `can_create_thread` is computed from `member` or `thread_create` — any org member can create threads, as can any app identity that has been granted the `thread:create` installation permission.

See [Organizations — Members Management](organizations.md#members-management) for how these permissions govern membership operations.

### Cluster Permissions

The authorization model includes a `cluster` type with a singleton object `cluster:global`:

| Permission | Capabilities |
|------------|-------------|
| **admin** | Register and manage cluster-scoped apps and runners. Platform administration. Add members to any organization directly |

Cluster admin permissions are stored as relationship tuples:

```
identity:<userId>, admin, cluster:global
```

Two identities hold this relation, for two different reasons. A human claims it by signing in — see [Bootstrap](#bootstrap). The platform holds one of its own, used to create the resources a release ships; see [Platform Resource Provisioning](operations/platform-provisioning.md).

### App Installation Permissions

Apps declare the permissions they need in their definition. When an app is installed into an organization, the Apps Service writes one OpenFGA tuple per declared permission, plus a membership tuple written for every installation regardless of what the app declared:

| App permission | Tuple written |
|----------------|---------------|
| *(every installation)* | `identity:<app_identity_id>, member, organization:<org_id>` |
| `thread:create` | `identity:<app_identity_id>, thread_create, organization:<org_id>` |
| `thread:write` | `identity:<app_identity_id>, thread_write, organization:<org_id>` |
| `participant:add` | `identity:<app_identity_id>, participant_add, organization:<org_id>` |
| `inbox:write` | `identity:<app_identity_id>, inbox_write, organization:<org_id>` |

The membership tuple is the same relation a user's membership writes, so an installed app satisfies every relation computed from `member` — including `can_initiate` on an `internal` agent, which resolves through `member from internal_access` and is reachable no other way. It is written by the Apps Service directly; no [`memberships`](organizations.md#members-management) row is created, because that table models people joining organizations. See [Apps — Organization Membership](apps.md#organization-membership).

These tuples flow into computed relations:
- An app with `thread_write` on an org satisfies `can_write` on any thread in that org (via `thread_write from org` in the thread type).
- An app with `participant_add` on an org satisfies `can_add_participant` on any thread in that org.
- An app with `thread_create` on an org satisfies `can_create_thread` on the org.
- An app with `inbox_write` on an org satisfies `can_write_inbox` on any [`agent_instance`](#agent_instance) in that org.

When the app is uninstalled, the Apps Service deletes all tuples written at install time.

See [Apps — Permissions](apps.md#permissions) for the permission vocabulary.

## Tuple Lifecycle

Services own the tuples for the resources they manage. Tuples are written and deleted synchronously with the state changes that drive them.

| Event | Tuple written | Written by |
|-------|---------------|-----------|
| Organization membership becomes active | `identity:<id>, <role>, organization:<org_id>` | Organizations |
| Organization membership removed | Delete `identity:<id>, <role>, organization:<org_id>` | Organizations |
| Membership role updated | Delete old role tuple, write new | Organizations |
| Model created | `organization:<org_id>, org, model:<model_id>` | LLM Service |
| Model deleted | Delete `organization:<org_id>, org, model:<model_id>` | LLM Service |
| Agent created | `organization:<org_id>, org, agent:<id>`; `identity:<creator>, owner, agent:<id>`; if `availability=internal`: `organization:<org_id>, internal_access, agent:<id>` | Agents |
| Agent instance created | `agent:<class_id>, class, agent_instance:<id>`; `organization:<org_id>, org, agent_instance:<id>` | Agents |
| Agent instance deleted (terminated) | Delete all tuples on `agent_instance:<id>` | Agents |
| Sandbox created | `organization:<org_id>, org, sandbox:<id>`; `identity:<creator_id>, owner, sandbox:<id>` | Agents |
| Sandbox shared / unshared | `identity:<target_id>, collaborator, sandbox:<id>` (or `group:<group_id>#member`), written on share and deleted on unshare | Agents |
| Sandbox hard-purged (retention policy; not on soft-`terminated`) | Delete all tuples on `sandbox:<id>` | Agents |
| Agent availability flipped `private → internal` | `organization:<org_id>, internal_access, agent:<id>` | Agents |
| Agent availability flipped `internal → private` | Delete `organization:<org_id>, internal_access, agent:<id>` | Agents |
| `SetAgentRole(identity, role)` | `identity:<id>, <role>, agent:<agent_id>`; if the identity previously held a different role on the agent: delete `identity:<id>, <old_role>, agent:<agent_id>` | Agents |
| `SetAgentRole(group, role)` | `group:<id>#member, <role>, agent:<agent_id>`; old role tuple removed if any | Agents |
| `RemoveAgentRole(identity)` | Delete `identity:<id>, <role>, agent:<agent_id>` | Agents |
| Agent deleted | Delete all tuples on `agent:<id>` (`org`, `internal_access`, every role tuple) | Agents |
| Group created | `group:<id>, org, organization:<org_id>` | Groups |
| Member added to group | `identity:<member_id>, member, group:<id>` | Groups |
| Member removed from group | Delete `identity:<member_id>, member, group:<id>` | Groups |
| Group deleted | Delete all tuples on `group:<id>` (`org`, every `member`, every `admin`); other services delete grants referencing `group:<id>#member` on consumption of `group.updated` | Groups |
| Thread created | `organization:<org_id>, org, thread:<thread_id>` + participant tuples | Threads |
| Participant added to thread | `identity:<id>, participant, thread:<thread_id>` | Threads |
| App installed | Permission tuples per declared permission (see above) | Apps Service |
| App uninstalled | Delete all install-time permission tuples | Apps Service |
| Cluster admin granted | `identity:<id>, admin, cluster:global` | Users (via gateway) / bootstrap |
| Cluster admin revoked | Delete `identity:<id>, admin, cluster:global` | Users (via gateway) |

Resources that are deleted (threads archived, organizations deleted) do not require explicit tuple cleanup beyond what is listed above — thread tuples become orphaned but are harmless since the thread record no longer exists to authorize against. For organizations, membership tuples are deleted as part of the deletion flow.

## Per-Service Authorization Reference

The tables below cover both Gateway-exposed operations (authorized via OpenFGA against the caller's identity) and internal-only operations (authorized via Istio — see [Internal RPC Authorization](#internal-rpc-authorization) for the enforcement model). Rows tagged `(via Gateway)` and `(internal)` distinguish the two paths when the same RPC name serves both call sites.

### Internal RPC Authorization

Internal-only RPCs are not exposed through the [Gateway](gateway.md). They are gated by two layers, in order:

1. **Istio mTLS** — proves the caller is a verified in-cluster service.
2. **Istio `AuthorizationPolicy`** — restricts each method to a specific allowlist of Kubernetes ServiceAccounts. Every internal RPC has an explicit policy; methods without a policy are not callable internally.

Internal RPCs do **not** consult OpenFGA, do **not** read `x-identity-id` from gRPC metadata, and do **not** require the caller to act on behalf of a user. The [Agents Orchestrator](agents-orchestrator.md), [LLM Proxy](llm-proxy.md), and other infrastructure services call these methods over Istio without injecting an identity header — they are recognized by their ServiceAccount, not by a platform identity.

Reconciliation, metering, and workload assembly are internal-only paths gated by `AuthorizationPolicy`, and the Orchestrator calls them with no identity header at all — Runners and Agents serve an absent `x-identity-id` as a platform call, which is also what lets it list the whole cluster in one request rather than once per organization.

#### When a platform service does present an identity

A few callees authorize a caller rather than serving an absent one, and the Orchestrator reaches those as **itself**: the [platform identity](identity-service.md), configured as `PLATFORM_IDENTITY_ID` and holding `admin` on `cluster:global`. Three paths need it:

| Callee | Why an identity is required |
|--------|------------------------------|
| [Notifications](notifications.md#platform-callers) `Subscribe` | Rooms are authorized per caller; there is no absent-identity path. |
| [Agents](agents-service.md) `GetUnackedInboxItems`, `GetUnackedInboxCount` | Instance-scoped reads. The Orchestrator must know which thread an instance was handed work for before it can place a workload. |
| [Groups](groups-service.md) `ListMemberGroups` | Reads an agent's groups to resolve the role attributes its workload carries. |

Each verifies the claim against `admin` on `cluster:global` rather than trusting the `x-identity-type: platform` header, so a caller that sets the header without holding the relation is refused.

**This replaced impersonation.** The Orchestrator previously sent an agent instance's or agent's own `identity_id` as the caller on these paths — and, for anything organization-scoped, elected the first agent it found in each organization and borrowed that one. It was a platform service claiming to be one of the things it manages, in order to satisfy checks written for that thing. It also could not work for `sandbox_org:`, which wants `can_list_sandboxes`: an instance is a `member` of its organization and holds no such relation, so that subscription was refused on every attempt.

The grants are read-only. Writes stay with the principal who owns them — Groups' create/update/delete remain owner-gated, and `AckInboxItems` remains the instance's alone.

The [Ziti Management Service](#ziti-management-service) is the canonical example of this pattern. The same model applies to internal methods on Runners, Agents, Threads, Secrets, LLM, and Notifications listed in the per-service tables below.

### Users Service

| Operation | Check |
|-----------|-------|
| `GetMe` | Authenticated (no OpenFGA check; returns caller's own profile) |
| `CreateUser`, `GetUser`, `GetUserByOIDCSubject`, `ListUsers`, `UpdateUser`, `DeleteUser` | `admin` on `cluster:global` |
| `CreateAPIToken`, `ListAPITokens`, `RevokeAPIToken` | Self only (`identity_id` from request context == token owner) |
| `CreateDevice`, `ListDevices`, `DeleteDevice` | Self only |

### Organizations Service

| Operation | Check |
|-----------|-------|
| `CreateOrganization` | Any authenticated identity (creator becomes owner) |
| `GetOrganization` | `member` on `organization:<id>` |
| `ListOrganizations` | Returns organizations where caller is a `member` (uses `ListObjects`), or every organization if caller has `admin` on `cluster:global` |
| `UpdateOrganization`, `DeleteOrganization` | `owner` on `organization:<id>` |
| `CreateMembership` | `can_add_member` on `organization:<id>` (→ active) or `can_invite` (→ pending) |
| `AcceptMembership`, `DeclineMembership` | Self only (invitee's `identity_id` matches the membership target) |
| `RemoveMembership`, `UpdateMembershipRole`, `ListMembers` | `can_manage_members` on `organization:<id>` |
| `ListMyMemberships` | Self only (returns caller's own memberships) |

### Agents Service

All agent resources (Agents, Environments, MCPs, Skills, ENVs, InitScripts, Volumes) are org-scoped. Sub-resources inherit their parent's organization. Agent- and environment-level access is split between org membership (metadata visibility and creation) and per-resource roles (configuration access, availability changes, role management) — see the [`agent`](#agent) and [`environment`](#environment) OpenFGA types.

| Operation | Check |
|-----------|-------|
| `CreateAgent` | `owner` on `organization:<org_id>` |
| `ListAgents`, `GetAgent` (metadata fields only), `ListEnvironments`, `GetEnvironment` (metadata fields only) | `member` on `organization:<org_id>` |
| `GetAgent` (configuration fields), `ListMCPs`, `ListSkills`, `ListENVs`, `ListInitScripts`, and the `Get` counterpart of each sub-resource, scoped to an agent | `can_read_config` on `agent:<agent_id>` |
| `UpdateAgent` on configuration fields; Create / Update / Delete on any agent sub-resource (MCP, Skill, ENV, InitScript) | `can_edit_config` on `agent:<agent_id>` |
| `UpdateAgent` on the `environment_id` field, `CreateAgent` with an `environment_id` | `can_edit_config` on `agent:<agent_id>` **and** `can_use` on `environment:<environment_id>` |
| `UpdateAgent` on the `availability` field, `DeleteAgent` | `can_delete` on `agent:<agent_id>` |
| `SetAgentRole`, `RemoveAgentRole`, `ListAgentRoles` | `can_manage_roles` on `agent:<agent_id>`; `SetAgentRole` additionally requires the target identity to satisfy `member` on the agent's organization |
| `ListMyAgentRoles` | Self only — returns the caller's own role assignments |
| `CreateEnvironment` | `can_create_environment` on `organization:<org_id>`; the creator is written as `owner` |
| `GetEnvironment` (configuration fields), `ListVolumes`, `ListMCPs`, `ListInitScripts`, `ListENVs` scoped to an environment, and their `Get` counterparts | `can_read_config` on `environment:<environment_id>` |
| `UpdateEnvironment` on configuration fields; Create / Update / Delete on any environment sub-resource (Volume, MCP, InitScript, ENV) | `can_edit_config` on `environment:<environment_id>` |
| `UpdateEnvironment` on the `availability` field, `DeleteEnvironment` | `can_delete` on `environment:<environment_id>` |
| `SetEnvironmentRole`, `RemoveEnvironmentRole`, `ListEnvironmentRoles` | `can_manage_roles` on `environment:<environment_id>`; `SetEnvironmentRole` additionally requires the target identity to satisfy `member` on the environment's organization |
| `CreateInstance` | `can_initiate` on `agent:<class_id>` (same gate as adding the class to a thread). Also called internally by Threads during the [class-on-add rewrite](threads.md#class-on-add-rewrite) |
| `GetInstance`, `ListInstances` (via Gateway) | `member` on `organization:<org_id>` |
| `ListInstances` (internal) | Internal only (Orchestrator via Istio) — desired-state query (`state=active, has_unacked=true`) across organizations |
| `PauseInstance`, `ResumeInstance`, `DeleteInstance` | `can_manage` on `agent_instance:<id>`; also called internally by the Orchestrator (via Istio) for failure-driven pauses |
| `WriteInboxItem` | `can_write_inbox` on `agent_instance:<id>` |
| `FanoutInboxItem` | Internal only (Threads via Istio) — thread participation already enforced by Threads |
| `GetUnackedInboxItems`, `AckInboxItems`, `GetUnackedInboxCount` | Self only — `caller.identity_id == agent_instance_id` (no OpenFGA check) |
| Get, List (any resource, internal) | Internal only (Orchestrator via Istio) — used by [workload spec assembly](agents-orchestrator.md#workload-spec-assembly); returns resolved sub-resources across organizations without an org or per-agent check |
| `ResolveAgentIdentity` | Internal only (Tracing via Istio) |
| `CreateSandbox` | `can_create_sandbox` on `organization:<org_id>` **and** `can_use` on `environment:<environment_id>`; owner is derived from authenticated context |
| `GetSandbox` | `can_read` on `sandbox:<id>` (owner, collaborator, or org owner) |
| `ListSandboxes` (own) | `member` on `organization:<org_id>` and server filters to sandboxes the caller owns or collaborates on |
| `ListSandboxes` (`all=true`) | `can_list_sandboxes` on `organization:<org_id>` |
| `StopSandbox`, `DeleteSandbox` | `can_stop` / `can_delete` on `sandbox:<id>` |
| `EnsureSandboxRunning` | `can_connect` on `sandbox:<id>` |
| `ShareSandbox`, `UnshareSandbox`, `ListSandboxShares` | `can_share` on `sandbox:<id>` (the owner); `ShareSandbox` additionally requires the target identity to satisfy `member` on the sandbox's organization |
| `UpdateSandboxLastSession` | Internal only (Terminal Proxy via Istio) |

Agent workload identities (`identity_type == "agent_instance"`) satisfy `member` on their organization — resolved through the instance's `org` relation on the [`agent_instance` type](#agent_instance) — and may call read APIs needed for self-configuration, including `ListENVs` against their class (via the instance's `class` relation). `ListENVs` never returns resolved secret values — secret-backed ENVs expose only the `secret_id` reference. The Orchestrator injects all ENV values (plain-text and resolved secrets) as container environment variables at assembly time. The per-agent role model gates access by other identities and does not alter instance self-read.

### Runners Service

Runners can be cluster-scoped (`organization_id` null) or org-scoped.

| Operation | Check |
|-----------|-------|
| `RegisterRunner` (cluster-scoped) | `admin` on `cluster:global` |
| `RegisterRunner` (org-scoped) | `owner` on `organization:<org_id>` |
| `GetRunner`, `ListRunners` (via Gateway) | `member` on `organization:<org_id>` for org-scoped runners; any authenticated identity for cluster-scoped runners |
| `GetRunner` (internal) | Internal only (Orchestrator via Istio) — used by [runner selection](agents-orchestrator.md#runner-selection); returns the runner regardless of scope |
| `UpdateRunner`, `DeleteRunner` (cluster-scoped) | `admin` on `cluster:global` |
| `UpdateRunner`, `DeleteRunner` (org-scoped) | `owner` on `organization:<org_id>` |
| `EnrollRunner` | Runner's own identity (service token verification, not OpenFGA) |
| `CreateWorkload`, `UpdateWorkload`, `BatchUpdateWorkloadSampledAt` | Internal only (Orchestrator via Istio) |
| `ListWorkloads` (via Gateway) | `can_view_workloads` on `organization:<org_id>` (required request parameter) |
| `ListWorkloads` (internal) | Internal only (Orchestrator via Istio) — supports `runner_id_in`, `pending_sample`, and `status_in` filters across organizations; `organization_id` not required. Used by [workload reconciliation](agents-orchestrator.md#workload-reconciliation) and the [metering sampling loop](agents-orchestrator.md#sampling-algorithm) |
| `GetWorkload`, `StreamWorkloadLogs` | `can_view_workloads` on `organization:<workload.org_id>` |
| `ListWorkloadsByAgentInstance` (via Gateway) | `member` on `organization:<workload.org_id>` |
| `ListWorkloadsByAgentInstance` (internal) | Internal only (Orchestrator via Istio) — used by the [start decision](agents-orchestrator.md#start-decision) |
| `TouchWorkload` | For `owner_kind=agent_instance`: agent instance's own identity (`workload.owner_id == caller.identity_id`). For `owner_kind=sandbox`: internal only from Terminal Proxy via Istio after terminal ticket validation |
| `CreateVolume`, `UpdateVolume`, `BatchUpdateVolumeSampledAt` | Internal only (Orchestrator via Istio) |
| `ListVolumes` (via Gateway) | `can_view_volumes` on `organization:<org_id>` (required request parameter) |
| `ListVolumes` (internal) | Internal only (Orchestrator via Istio) — supports `runner_id_in`, `pending_sample`, and `status_in` filters across organizations; `organization_id` not required. Used by [volume reconciliation](agents-orchestrator.md#volume-reconciliation) and the [metering sampling loop](agents-orchestrator.md#sampling-algorithm) |
| `GetVolume` | `can_view_volumes` on `organization:<volume.org_id>` |
| `ListVolumesByAgentInstance` (via Gateway) | `member` on `organization:<volume.org_id>` |
| `ListVolumesByAgentInstance` (internal) | Internal only (Orchestrator via Istio) — used by [runner selection](agents-orchestrator.md#runner-selection) |

### Threads Service

| Operation | Check |
|-----------|-------|
| `CreateThread` | `can_create_thread` on `organization:<org_id>` AND for each agent participant (class or instance): `can_initiate` on `agent:<class_id>` (class resolved from the instance when an instance is passed) |
| `ArchiveThread` | `participant` on `thread:<id>` or `owner` on `organization:<thread.org_id>` |
| `DegradeThread` | Internal only (Orchestrator via Istio) — reserved for unrecoverable thread-level conditions. Instance-scoped failures (start failures, volume loss, runner deprovisioned) pause the instance instead — see [Agent Instances — Lifecycle](agent-instances.md#lifecycle) |
| `AddParticipant` | `can_add_participant` on `thread:<id>` AND if the participant is an agent class or instance: `can_initiate` on `agent:<class_id>` |
| `SendMessage` | `can_write` on `thread:<id>` |
| `GetThreads` | No OpenFGA check — returns threads where `caller.identity_id` is a participant (DB filter) |
| `ListOrganizationThreads` | `can_view_threads` on `organization:<org_id>` |
| `GetMessages` | `can_read` on `thread:<id>` |
| `GetUnackedMessages` | Self only — returns unacked messages for `caller.identity_id` as participant |
| `GetUnackedMessageCounts` | Self only — returns per-thread counts for `caller.identity_id` as participant |
| `AckMessages` | Self only — caller must be the recipient |

### Chat Service

Chat wraps Threads. Thread-level authorization checks apply, including the per-agent [availability check](threads.md#agent-availability-check) on creation and participant addition.

| Operation | Check |
|-----------|-------|
| `CreateChat` | `can_create_thread` on `organization:<org_id>` AND for each agent participant: `can_initiate` on `agent:<participant_id>` |
| `GetChats` | No OpenFGA check — returns chats where caller is a participant (DB filter) |
| `GetMessages` | `can_read` on `thread:<id>` |
| `SendMessage` | `can_write` on `thread:<id>` |
| `MarkAsRead` | Self only — caller must be a participant |

### Files Service

Files are org-scoped. Access is determined by organization membership. No separate OpenFGA type is needed.

| Operation | Check |
|-----------|-------|
| `UploadFile` | `member` on `organization:<org_id>` |
| `GetFileMetadata`, `GetDownloadURL`, `GetFileContent` | `member` on `organization:<file.org_id>` |
| `DeleteFile` | File uploader (`file.uploader_id == caller.identity_id`) or `owner` on `organization:<file.org_id>` |

### Secrets Service

| Operation | Check |
|-----------|-------|
| Create, Update, Delete (providers, secrets) | `owner` on `organization:<org_id>` |
| Get, List (providers, secrets) | `member` on `organization:<org_id>` |
| `ResolveSecretValue` (via Gateway) | `admin` on `cluster:global` |
| `ResolveSecretValue` (internal) | Internal only (Orchestrator via Istio) |

### Images Service

[Image](resource-definitions.md#image) visibility affects who can read image records: `public` images are visible to any authenticated identity; `internal` images only to members of the owning organization. Visibility also governs use — an organization that can read an image can build environments on it, and there is no separate grant.

| Operation | Check |
|-----------|-------|
| `CreateImage`, `UpdateImage`, `DeleteImage` | `owner` on `organization:<org_id>` |
| `GetImage`, `ListImages`, `ListVersions`, `RefreshImage` | `member` on `organization:<org_id>`, or the image is `public` |
| `ResolveVersion` | Internal only (Agents Service, Image Proxy via Istio) |
| `CountImagesReferencingSecret` | Internal only (Secrets Service via Istio) |

No `image` OpenFGA type is introduced — both visibility values resolve against existing organization relations, so images need no per-resource tuples.

`secret_id` is checked separately from the operation: the secret an image names must belong to the image's organization. Owner on the organization is not enough, because it would otherwise let an owner name a secret they cannot read and have discovery send its value to a registry they choose.

### LLM Service

| Operation | Check |
|-----------|-------|
| Create, Update, Delete (providers, models) | `owner` on `organization:<org_id>` |
| Get, List (providers, models) | `member` on `organization:<org_id>` |
| `ResolveModel` | Internal only (LLM Proxy via Istio) |

Model access at call time (checked by the LLM Proxy, not the LLM Service):

| Operation | Check |
|-----------|-------|
| Use a model (LLM API call) | `can_use` on `model:<model_id>` |

### Apps Service

App visibility affects who can read app records: `public` apps are visible to any authenticated identity; `internal` apps are visible only to members of the owning organization.

| Operation | Check |
|-----------|-------|
| `CreateApp` | `owner` on `organization:<org_id>` (owning org) |
| `GetApp`, `GetAppBySlug` (public app) | Any authenticated identity |
| `GetApp`, `GetAppBySlug` (internal app) | `member` on `organization:<app.org_id>` |
| `ListApps` | Returns public apps (any authenticated) + own-org apps (`member` on org, via filter) |
| `UpdateApp`, `DeleteApp` | `owner` on `organization:<app.org_id>` |
| `GetAppProfile` | Any authenticated identity (used by Chat for message display) |
| `InstallApp` | `owner` on `organization:<install_org_id>` (installing org) + app visibility allows it |
| `GetInstallation`, `ListInstallations` | `member` on `organization:<install_org_id>` |
| `GetInstallationByIdentityId` | Internal (Gateway app proxy hot path, authenticated) |
| `UpdateInstallation`, `UninstallApp` | `owner` on `organization:<install_org_id>` |
| `GetInstallationConfiguration` | App's own identity (`caller.identity_id == installation.app.identity_id`) |

### Tracing Service

| Operation | Check |
|-----------|-------|
| `Export` (span ingestion via OpenZiti) | Any agent with a valid OpenZiti identity (network-level auth, no OpenFGA check) |
| `agyn.thread.id` attribute verification | `can_read` on `thread:<thread_id>` |
| `ListSpans` | `member` on `organization:<org_id>` (required request parameter) |
| `GetTrace`, `GetSpan` | `member` on `organization:<span.org_id>` (resolved from stored `agyn.organization.id`) |

### Expose Service

| Operation | Check |
|-----------|-------|
| `AddExposure` (standard: no `workload_id` in request) | Caller is the workload — `x-workload-id` present (injected by the Gateway from the verified OpenZiti connection) and resolving to a live workload |
| `AddExposure` (explicit `workload_id`, sandbox-owned workload) | `can_connect` on `sandbox:<workload.owner_id>` |
| `AddExposure` (explicit `workload_id`, any other owner) | `admin` on `cluster:global` |
| `RemoveExposure` | Caller is the workload; `can_connect` on `sandbox:<exposure.owner_id>` when sandbox-owned; or `owner` on `organization:<exposure.organization_id>` |
| `ListExposures` | Caller is the workload, or `member` on `organization:<exposure.organization_id>` |

`can_connect` is the relation the [Terminal Proxy](terminal-proxy.md#authorization) already checks, used here for the same reason: a shell can run `agyn expose` itself, so managing a sandbox's ports from the [Sandboxes app](sandboxes-app.md) confers nothing a session does not. It introduces no relation and no tuple — `sandbox` already defines `can_connect`.

The self-service checks are identity equality against the workload, not relations on its owner, and they read the same for an [agent instance](#agent_instance) and a [sandbox](#sandbox). Who may *reach* an exposed port is a separate question, answered today by the `#all` Dial policy rather than by this model — see [Expose Service — Authorization](expose-service.md#authorization).

### Networks Service

Networks, Tunnels, PrivateResources, and PrivateResourceAccess grants are all organization-scoped resources managed by the [Networks service](networks-service.md). No new OpenFGA types are introduced — checks rely on existing organization-level relations and on `can_edit_config` on the [`agent`](#agent) or [`environment`](#environment) a grant targets.

An [environment principal](private-networks.md#why-an-environment-is-a-principal) reaches every workload running that environment, including a sandbox any holder of `can_use` may start. `can_edit_config` is the right gate for it — the same one that governs the environment's ENVs, volumes, and egress rule attachments, all of which a shell in such a sandbox already reaches. Granting a private resource to an environment adds a destination to that set; it does not widen who is standing in front of it.

| Operation | Check |
|-----------|-------|
| `CreateNetwork`, `UpdateNetwork`, `DeleteNetwork` | `owner` on `organization:<org_id>` |
| `GetNetwork`, `ListNetworks` | `member` on `organization:<org_id>` |
| `CreateTunnelCredential`, `DeleteTunnelCredential` | `owner` on `organization:<org_id>` |
| `GetTunnelCredential`, `ListTunnelCredentials` | `member` on `organization:<org_id>` |
| `CreatePrivateResource`, `UpdatePrivateResource`, `DeletePrivateResource` | `owner` on `organization:<org_id>` |
| `GetPrivateResource`, `ListPrivateResources` | `member` on `organization:<org_id>` |
| `CreatePrivateResourceAccess` (`agent` principal) | `can_edit_config` on `agent:<agent_id>` + cross-org guard (`organization:<resource.org_id>` holds `org` on `agent:<agent_id>`) |
| `CreatePrivateResourceAccess` (`environment` principal) | `can_edit_config` on `environment:<environment_id>` + cross-org guard (`organization:<resource.org_id>` holds `org` on `environment:<environment_id>`) |
| `CreatePrivateResourceAccess` (`user`, `app`, or `group` principal) | `owner` on `organization:<resource.org_id>` + cross-org guard (`organization:<resource.org_id>` holds `org` on the principal) |
| `DeletePrivateResourceAccess` | Same check as the corresponding `CreatePrivateResourceAccess` |
| `ListPrivateResourceAccess` | `member` on `organization:<org_id>` |

### Groups Service

| Operation | Check |
|-----------|-------|
| `CreateGroup`, `UpdateGroup`, `DeleteGroup` | `owner` on `organization:<org_id>` |
| `GetGroup`, `ListGroups` | `member` on `organization:<org_id>` |
| `AddMember`, `RemoveMember` | `can_edit` on `group:<group_id>` (group `admin` OR org `owner`) + cross-org guard (member belongs to the same org) |
| `ListMembers` | `can_view` on `group:<group_id>` (group `member` OR org `member`) |
| `ListMemberGroups` (other identity) | `member` on `organization:<org_id>` |
| `ListMemberGroups` (self) | Authenticated (caller may list their own group memberships) |
| `ListMemberGroupsBatch` (internal) | Internal only (Agents Orchestrator / Networks service via Istio) |

### EgressRules Service

| Operation | Check |
|-----------|-------|
| `CreateEgressRule`, `UpdateEgressRule`, `DeleteEgressRule` | `owner` on `organization:<org_id>` |
| `GetEgressRule`, `ListEgressRules` | `member` on `organization:<org_id>` |
| `CreateEgressRuleAttachment`, `DeleteEgressRuleAttachment` (agent target) | `can_edit_config` on `agent:<agent_id>`. On create, the service additionally verifies the agent belongs to the rule's organization (a `Check` that `organization:<rule.org_id>` holds the `org` relation on `agent:<agent_id>`) to prevent cross-org attachments |
| `CreateEgressRuleAttachment`, `DeleteEgressRuleAttachment` (environment target) | `can_edit_config` on `environment:<environment_id>`, with the same cross-org guard against the `org` relation on the environment |
| `ListEgressRuleAttachments` (by `agent_id` / `environment_id`) | `can_read_config` on that target |
| `ListEgressRuleAttachments` (by `rule_id`) | `member` on `organization:<rule.org_id>` |
| `ListEgressRulesByAgent` (internal) | Internal only (Egress Gateway via Istio) |
| `CountRulesReferencingSecret` (internal) | Internal only (Secrets service via Istio) |

No new OpenFGA types are introduced. Rules use existing organization-level checks; each attachment uses its target's `can_edit_config` / `can_read_config` — from the [agent](#agent) type or the [environment](#environment) type — plus that type's `org` relation for the cross-org guard.

### Notifications Service

The internal `Publish` RPC is Istio-only (trusted internal services). The external `Subscribe` (a ConnectRPC server stream through the [Gateway](gateway.md)) validates room access per subscription:

| Room pattern | Access check |
|--------------|-------------|
| `thread_participant:{id}` | `id == caller.identity_id` (identity equality, no OpenFGA). `thread_participant:me` is rewritten to `thread_participant:{caller.identity_id}` before this check — see [Notifications — Self-Subscription Sentinel](notifications.md#self-subscription-sentinel). |
| `instance_inbox:{id}` | `id == caller.identity_id` AND `caller.identity_type == agent_instance` (identity equality, no OpenFGA). `instance_inbox:me` is rewritten before the check. Only the instance itself may subscribe to its inbox room |
| `workload:{id}` | `member` on `organization:<workload.org_id>` |
| `agent:{id}` | `member` on `organization:<agent.org_id>` |
| `environment:{id}` | `member` on `organization:<environment.org_id>`. Carries `environment.updated`, which every agent and sandbox running the environment is affected by |
| `agent_instance:{id}` | `member` on `organization:<instance.org_id>`. Carries `instance.updated`. An instance's own identity satisfies `member` through its `org` relation, which is how the Orchestrator watches the instances it reconciles |
| `sandbox_owner:{owner_id}` | `owner_id == caller.identity_id` (identity equality, no OpenFGA). `:me` is not accepted for this room pattern |
| `sandbox:{sandbox_id}` | `can_read` on `sandbox:<sandbox_id>`. Carries one sandbox, so collaborators — who the owner-keyed room cannot reach — receive its updates |
| `sandbox_org:{organization_id}` | `can_list_sandboxes` on `organization:<organization_id>` |
| `trace:{trace_id}` | `member` on `organization:<trace.org_id>` |

### Metering Service

Internal only. Producers are internal services (LLM Proxy, Orchestrator, Threads) communicating via Istio. No OpenFGA checks.

### Ziti Management Service

Internal only (Istio). Not exposed through the Gateway. Istio `AuthorizationPolicy` is the enforcement mechanism — only specific service accounts (Orchestrator, Runners service, Agents service, Gateway, LLM Proxy, Tracing service) may call `ZitiManagement`. Application-level guards prevent callers from requesting identity role attributes beyond their own scope (e.g., only the Orchestrator may call `CreateAgentIdentity`).

## How Services Use Authorization

### Permission Checks

Before performing an operation, a service calls `Check` on the Authorization service:

```
Check(identity:<identity_id>, can_read, thread:<thread_id>) → allowed: bool
```

If denied, the service returns a permission error. The identity is available in gRPC metadata (see [Authentication](authn.md)). Organization context, when needed, is passed as a request parameter.

### Relationship Writes

When state changes, the owning service writes relationship tuples through the Authorization service's `Write` method. `Write` supports atomic multi-tuple writes (adds and deletes in a single call).

## Model Deployment

The authorization model is managed as infrastructure-as-code in the Authorization service repo (`agynio/authorization`), under `terraform/`:

| Content | Description |
|---------|-------------|
| Authorization model DSL | Type definitions, relations, computed permissions |
| Terraform module | Creates the OpenFGA store, writes the model version |
| Model tests | `fga model test` — validates expected check results against sample tuples |

Terraform is applied directly from `agynio/authorization/terraform/` to manage the OpenFGA store and model versions.

The [OpenFGA Terraform provider](https://registry.terraform.io/providers/openfga/openfga/latest) manages store and model lifecycle:

- `openfga_store` — creates the store, persists `store_id` in Terraform state.
- `openfga_authorization_model_document` — produces a stable JSON representation from the DSL. Output changes only on semantic changes (not formatting).
- `openfga_authorization_model` — writes a new model version. OpenFGA supports model versioning natively — each write creates a new version, old tuples continue to work.

Terraform outputs (`store_id`, `model_id`) feed into the Authorization service's configuration (environment variables via K8s config).

### Model Update Flow

1. Change the authorization model DSL in `agynio/authorization/terraform/`.
2. Run `fga model test` to validate.
3. Merge.
4. `terraform apply` from `agynio/authorization/terraform/` writes the new model version to OpenFGA.
5. Authorization service picks up the new `model_id` on next deploy (or uses latest if not pinned).

## OpenFGA Deployment

OpenFGA runs as a service within the Kubernetes cluster. It uses PostgreSQL as its data store (same infrastructure, separate database or schema).

| Aspect | Details |
|--------|---------|
| Storage | PostgreSQL |
| Protocol | gRPC |
| Deployment | Kubernetes (Helm chart) |
| Local development | Part of the bootstrap cluster |

## Bootstrap

Cluster admins are **declared by the install**, not claimed by arriving. A release names the people who hold the role, and the [provisioning controller](operations/platform-provisioning.md#cluster-administrators) grants `admin` on `cluster:global` to each named account once that account exists — which is to say once that person has signed in. Nobody is granted the role for being first, and an account nobody named never receives it.

An install that names no administrator therefore has none. That is deterministic and recoverable — declare one — rather than a race decided by who opened the Console first.

The controller itself needs the relation before it can grant anything, so the platform holds a **separate** cluster admin identity of its own, authenticated by a bootstrap token on the [Gateway](gateway.md) rather than through OIDC. It is not a user account and never becomes one. See [Platform Resource Provisioning](operations/platform-provisioning.md#the-platform-admin-identity).

No bootstrap path writes to a service's database directly. Every identity acquires the relation through the service that owns it.
