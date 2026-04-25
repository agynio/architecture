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
    define can_manage_members: owner
    define can_view_threads: owner or admin from cluster
    define can_view_workloads: owner or admin from cluster
    define can_view_volumes: owner or admin from cluster
    define can_add_member: admin from cluster
    define can_create_thread: member or thread_create
    define thread_create: [identity]
    define thread_write: [identity]
    define participant_add: [identity]
```

`owner` implies `member`, `can_invite`, `can_manage_members`, `can_view_threads`, `can_view_workloads`, and `can_view_volumes`. `owner` does **not** imply `can_create_thread` directly — instead `can_create_thread` is computed from `member`, and owners are also members, so owners can always create threads.

`thread_create`, `thread_write`, and `participant_add` are **app installation permissions** — direct relations written when an app is installed. See [App Installation Permissions](#app-installation-permissions).

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
| **thread_create** | app permission | Written for app installations that declare the `thread:create` permission |
| **thread_write** | app permission | Written for app installations that declare the `thread:write` permission |
| **participant_add** | app permission | Written for app installations that declare the `participant:add` permission |

#### Computed relations

- `owner` implies `member`, `can_invite`, `can_manage_members`, `can_view_threads`, `can_view_workloads`, and `can_view_volumes`.
- `can_add_member`, `can_view_threads`, `can_view_workloads`, and `can_view_volumes` each include `admin from cluster` — any identity with the `admin` relation on `cluster:global` holds these permissions on every organization. Modeled as cross-type computed relations, not as explicit per-organization tuples.
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

See [Cluster Permissions — Bootstrap](#bootstrap) for how the initial admin is seeded.

### App Installation Permissions

Apps declare the permissions they need in their definition. When an app is installed into an organization, the Apps Service writes one OpenFGA tuple per declared permission:

| App permission | Tuple written |
|----------------|---------------|
| `thread:create` | `identity:<app_identity_id>, thread_create, organization:<org_id>` |
| `thread:write` | `identity:<app_identity_id>, thread_write, organization:<org_id>` |
| `participant:add` | `identity:<app_identity_id>, participant_add, organization:<org_id>` |

These tuples flow into thread-level computed relations:
- An app with `thread_write` on an org satisfies `can_write` on any thread in that org (via `thread_write from org` in the thread type).
- An app with `participant_add` on an org satisfies `can_add_participant` on any thread in that org.
- An app with `thread_create` on an org satisfies `can_create_thread` on the org.

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
| Thread created | `organization:<org_id>, org, thread:<thread_id>` + participant tuples | Threads |
| Participant added to thread | `identity:<id>, participant, thread:<thread_id>` | Threads |
| App installed | Permission tuples per declared permission (see above) | Apps Service |
| App uninstalled | Delete all install-time permission tuples | Apps Service |
| Cluster admin granted | `identity:<id>, admin, cluster:global` | Users (via gateway) / bootstrap |
| Cluster admin revoked | Delete `identity:<id>, admin, cluster:global` | Users (via gateway) |

Resources that are deleted (threads archived, organizations deleted) do not require explicit tuple cleanup beyond what is listed above — thread tuples become orphaned but are harmless since the thread record no longer exists to authorize against. For organizations, membership tuples are deleted as part of the deletion flow.

## Per-Service Authorization Reference

This table summarizes the authorization check applied to each Gateway-exposed operation. Internal-only operations (not exposed through the Gateway) use Istio mTLS for caller trust and do not require OpenFGA checks.

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

All agent resources (Agents, Volumes, MCPs, Skills, Hooks, ENVs, InitScripts, VolumeAttachments, ImagePullSecretAttachments) are org-scoped. Sub-resources inherit their parent's organization.

| Operation | Check |
|-----------|-------|
| Create, Update, Delete (any resource) | `owner` on `organization:<org_id>` |
| Get, List (any resource) | `member` on `organization:<org_id>` |
| `ResolveAgentIdentity` | Internal only (Istio) |

Agent workload identities (`identity_type == "agent"`) satisfy `member` and may call all read APIs including `ListENVs`. `ListENVs` never returns resolved secret values — secret-backed ENVs expose only the `secret_id` reference. The Orchestrator injects all ENV values (plain-text and resolved secrets) as container environment variables at assembly time.

### Runners Service

Runners can be cluster-scoped (`organization_id` null) or org-scoped.

| Operation | Check |
|-----------|-------|
| `RegisterRunner` (cluster-scoped) | `admin` on `cluster:global` |
| `RegisterRunner` (org-scoped) | `owner` on `organization:<org_id>` |
| `GetRunner`, `ListRunners` | `member` on `organization:<org_id>` for org-scoped runners; any authenticated identity for cluster-scoped runners |
| `UpdateRunner`, `DeleteRunner` (cluster-scoped) | `admin` on `cluster:global` |
| `UpdateRunner`, `DeleteRunner` (org-scoped) | `owner` on `organization:<org_id>` |
| `EnrollRunner` | Runner's own identity (service token verification, not OpenFGA) |
| `CreateWorkload`, `UpdateWorkload`, `BatchUpdateWorkloadSampledAt` | Internal only (Orchestrator via Istio) |
| `ListWorkloads` | `can_view_workloads` on `organization:<org_id>` (required request parameter) |
| `GetWorkload`, `StreamWorkloadLogs` | `can_view_workloads` on `organization:<workload.org_id>` |
| `ListWorkloadsByThread` | `member` on `organization:<workload.org_id>` |
| `TouchWorkload` | Agent's own identity (`workload.agent_identity_id == caller.identity_id`) |
| Volume operations (internal) | Internal only (Orchestrator via Istio) |
| `ListVolumes` | `can_view_volumes` on `organization:<org_id>` (required request parameter) |
| `GetVolume` | `can_view_volumes` on `organization:<volume.org_id>` |
| `ListVolumesByThread` | `member` on `organization:<volume.org_id>` |

### Threads Service

| Operation | Check |
|-----------|-------|
| `CreateThread` | `can_create_thread` on `organization:<org_id>` |
| `ArchiveThread` | `participant` on `thread:<id>` or `owner` on `organization:<thread.org_id>` |
| `AddParticipant` | `can_add_participant` on `thread:<id>` |
| `SendMessage` | `can_write` on `thread:<id>` |
| `GetThreads` | No OpenFGA check — returns threads where `caller.identity_id` is a participant (DB filter) |
| `ListOrganizationThreads` | `can_view_threads` on `organization:<org_id>` |
| `GetMessages` | `can_read` on `thread:<id>` |
| `GetUnackedMessages` | Self only — returns unacked messages for `caller.identity_id` as participant |
| `GetUnackedMessageCounts` | Self only — returns per-thread counts for `caller.identity_id` as participant |
| `AckMessages` | Self only — caller must be the recipient |

### Chat Service

Chat wraps Threads. Thread-level authorization checks apply.

| Operation | Check |
|-----------|-------|
| `CreateChat` | `can_create_thread` on `organization:<org_id>` |
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
| Create, Update, Delete (providers, secrets, image pull secrets) | `owner` on `organization:<org_id>` |
| Get, List (providers, secrets, image pull secrets) | `member` on `organization:<org_id>` |
| `ResolveSecretValue` (via Gateway) | `admin` on `cluster:global` |
| `ResolveSecretValue`, `ResolveImagePullSecret` (internal) | Internal only (Orchestrator via Istio) |

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
| `AddExposure` (standard: no `workload_id` in request) | Agent's own identity — `workload.agent_identity_id == caller.identity_id` (from `x-workload-id` header) |
| `AddExposure` (explicit `workload_id`) | `admin` on `cluster:global` |
| `RemoveExposure` | Agent's own identity or `owner` on `organization:<workload.org_id>` |
| `ListExposures` | Agent's own identity or `member` on `organization:<workload.org_id>` |

### Notifications Service

The internal `Publish` RPC is Istio-only (trusted internal services). The external `Subscribe` (Socket.IO) validates room access per subscription:

| Room pattern | Access check |
|--------------|-------------|
| `thread_participant:{id}` | `id == caller.identity_id` (identity equality, no OpenFGA) |
| `workload:{id}` | `can_view_workloads` on `organization:<workload.org_id>` |
| `agent:{id}` | `member` on `organization:<agent.org_id>` |
| `trace:{trace_id}` | `member` on `organization:<trace.org_id>` |

### Token Counting Service

| Operation | Check |
|-----------|-------|
| All counting operations | `member` on `organization:<org_id>` (counting is scoped per org) |

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

The initial cluster admin is seeded during platform bootstrap. Terraform writes directly to PostgreSQL:

1. Terraform creates a user record in the Users service database. This is a platform-only user — not associated with any OIDC identity.
2. Terraform registers the user's identity in the Identity service database.
3. Terraform creates an API token for this user (writes to `user_api_tokens` table — hash of the generated token).
4. Terraform writes the OpenFGA tuple: `identity:<userId>, admin, cluster:global`.

The generated API token is stored as a Terraform output (sensitive) and is used for cluster-level operations — registering cluster-scoped apps and runners.
