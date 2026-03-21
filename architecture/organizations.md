# Organizations

## Overview

The platform uses **organizations** as the grouping unit for configuration resources. An organization owns agents, LLM providers, models, secret providers, secrets, channels, and chats. Resources that belong to an organization have an `organization_id` field.

Not all resources belong to organizations. Threads, files, agent state, and workloads are **independent resources** — access to them is governed by [ReBAC permissions](authz.md) rather than organizational membership. This separation reflects the domain: conversations (threads) and runtime artifacts (state, workloads) connect participants across organizational boundaries, while configuration resources (agents, providers, secrets) are organizational infrastructure.

## Organization Model

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique organization identifier |
| `name` | string | Display name |
| `created_at` | timestamp | Creation time |

## Organizations Service

The Organizations service is a **control plane** service. It replaces the former Tenants service.

### Responsibilities

| Concern | Description |
|---------|-------------|
| **Organization CRUD** | Create, read, update, delete organizations |
| **List accessible organizations** | Return organizations an identity can access. Queries [Authorization](authz.md) for the list of organization IDs, then enriches with organization details from its own database |

### Data Store

PostgreSQL — `organizations` table.

## Organization Access

Organization access is managed through [Authorization](authz.md) (OpenFGA relationship tuples). See [Authorization — Organization Permissions](authz.md#organization-permissions) for the permission model.

### Organization Listing

When the UI needs to display an organization switcher, it calls the Organizations service `ListOrganizations`. The Organizations service queries Authorization (`ListObjects`) for accessible organization IDs, then returns full organization details from its own database.

## Identities and Organizations

Any identity — user, agent, channel, or runner — can have access to an organization. What an identity can do within an organization is determined by its [authorization relationships](authz.md), not by its type.

An identity can have access to multiple organizations. User-to-organization membership is managed entirely through OpenFGA relationship tuples — the [Users](users.md) service has no organization association.

See [Identity](identity.md) for the identity registry and [Authentication](authn.md) for how identity context is propagated.

## Resource Scoping

Resources are classified into two categories: **org-scoped** and **independent**.

### Org-Scoped Resources

Org-scoped resources belong to an organization. They have an `organization_id` field and are listed/queried within the context of an organization.

| Service | Resources | Notes |
|---------|-----------|-------|
| [Agents](agents-service.md) | Agents, Volumes | Direct `organization_id` on the resource |
| [Agents](agents-service.md) | MCPs, Skills, Hooks, ENVs, InitScripts, Volume Attachments | Inherit org scope through parent (agent, MCP, or hook). No `organization_id` column — org is resolved via the parent chain. Can be denormalized if query patterns require it |
| [LLM](llm.md) | LLM Providers, Models | `organization_id` on the resource |
| [Secrets](secrets.md) | Secret Providers, Secrets | `organization_id` on the resource |
| [Channels](channels.md) | Channel configuration | `organization_id` on the resource |
| [Chat](chat.md) | Chats | `organization_id` for listing chats within an organization. The underlying [thread](threads.md) is independent |

### Independent Resources

Independent resources have no `organization_id`. Access is governed by [ReBAC permissions](authz.md) on the resource itself or on a related resource.

| Service | Resources | Access Model |
|---------|-----------|-------------|
| [Threads](threads.md) | Threads, Messages, MessageRecipients | ReBAC permissions on the thread |
| [Files](media.md) | File metadata and objects | ReBAC permissions via the thread that references the file |
| [Agent State](agent/state.md) | Agent state records | Owned by agent (`agent_id`) via ReBAC |
| [Runner](runner.md) | Workloads | Owned by agent via ReBAC |
| [Identity](identity.md) | Identity records | System-wide, no org association |
| [Users](users.md) | User records and profiles | System-wide, no org association |
| [Notifications](notifications.md) | Room subscriptions | Room-based routing by resource ID, no org scoping |
| [Tracing](tracing.md) | Tracing data | Independent |

## Data Isolation

Org-scoped services include `organization_id` as a column on org-scoped tables. Queries filter by `organization_id` when listing resources within an organization.

Independent resources do not filter by organization. Access control is enforced through [Authorization](authz.md) checks on the specific resource or its parent.

Object storage (S3) keys are not prefixed by organization — files are independent resources identified by UUID.

## Request Flow

There is no organization header on requests. Services that operate on org-scoped resources accept `organization_id` as a request parameter on methods that need it (e.g., listing agents in an organization, creating a chat in an organization). The [Authorization](authz.md) model enforces that the caller has the appropriate relationship to the organization.

The [Gateway](gateway.md) authenticates the identity but does not validate organization membership — that responsibility belongs to the authorization model, checked by the service performing the operation.

## Future: System-Wide Providers

The current model scopes all providers (LLM, secrets) to organizations. A future extension may introduce **system-wide providers** — providers available to all organizations without per-org configuration. The mechanism (e.g., OpenFGA wildcard relationships, a dedicated `system` type) will be designed when the use case is concrete. The org-scoped model documented here is forward-compatible with this extension.
