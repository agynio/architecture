# Multi-Tenancy

## Overview

The platform supports multiple tenants. A **tenant** is an isolated organizational unit that owns all resources (agents, MCP servers, workspaces, models, threads, chats, etc.). Resources are scoped to a tenant — they are not visible or accessible across tenant boundaries.

The **Tenant service** manages tenant lifecycle and tenant membership. It is the source of truth for which tenants exist and which identities belong to them.

## Tenant Model

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique tenant identifier |
| `name` | string | Display name |
| `created_at` | timestamp | Creation time |

## Tenant Service

The Tenant service is a **control plane** service. It manages organizational desired state — tenant definitions and membership — and is not on the live data processing path.

### Responsibilities

| Concern | Description |
|---------|-------------|
| **Tenant CRUD** | Create, read, update, delete tenants |
| **Tenant membership** | Manage identity↔tenant relationships: add member, remove member, list members, assign roles |
| **Membership queries** | List tenants for an identity (used by Gateway for tenant selection after authentication) |
| **Authorization writes** | Write tenant-level relationship tuples to the [Authorization](authz.md) service when membership changes (e.g., `identity:<id>` is `owner` of `tenant:<tenantId>`) |

### Consumers

| Consumer | Usage |
|----------|-------|
| **Gateway** | After authentication, query tenant memberships for the identity to resolve the active tenant |
| **UI** | Tenant switcher, tenant settings, membership management |
| **Any identity** | Create tenants, manage membership (subject to [authorization](authz.md)) |

### Data Store

PostgreSQL — `tenants` table and `tenant_memberships` table.

**Tenant memberships:**

| Field | Type | Description |
|-------|------|-------------|
| `identity_id` | string (UUID) | The member identity |
| `tenant_id` | string (UUID) | The tenant |
| `role` | string | Tenant-level role (e.g., `owner`, `member`) |
| `created_at` | timestamp | When the membership was created |

## Identities and Tenants

Any identity — user, agent, channel, or runner — can be a member of a tenant. The identity type does not restrict tenant operations. What an identity can do within a tenant is determined by its relationships in the [authorization model](authz.md), not by its type.

The identity↔tenant relationship is many-to-many. An identity can belong to multiple tenants. For users, the active tenant is selected per session from the identity's tenant memberships.

Tenant membership is managed within the platform by the Tenant service, not in the IdP.

### Tenant Creation

Any authenticated identity can create a tenant. The identity that creates a tenant becomes its owner. The Tenant service writes the ownership relationship to the [Authorization](authz.md) service.

### Tenant Membership

The tenant owner can add other identities as members. The Tenant service writes membership relationship tuples to the [Authorization](authz.md) service when members are added or removed.

## Resource Scoping

Every resource in the platform belongs to a tenant. The `tenant_id` is present on all resource tables and used in all queries as a mandatory filter.

| Service | Scoped Resources |
|---------|-----------------|
| Teams | Agents, Volumes, Volume Attachments, MCPs, Skills, Hooks, ENVs, InitScripts |
| LLM | LLM Providers, Models |
| Secrets | Secret Providers, Secrets |
| Threads | Threads, Messages, MessageRecipients |
| Chat | Chats (thin layer over Threads — tenant scoping comes from Threads) |
| Files | File metadata and object keys |
| Agent State | Agent state records |
| Runner | Workloads are labeled with `tenant_id` |
| Notifications | Room subscriptions are implicitly scoped (room names include tenant-scoped resource IDs) |

## Data Isolation

All services that use PostgreSQL include `tenant_id` as a column on every tenant-scoped table. Queries always filter by `tenant_id`. There is no cross-tenant data access path.

Object storage (S3) keys are prefixed with `tenant_id` to partition files by tenant.

## Tenant Resolution

Every authenticated identity is associated with a tenant for each request. The tenant is resolved during authentication and propagated in request context to all downstream services.

For identities with a single tenant (typically agents, channels, runners), the tenant is fixed — determined at identity creation. For identities with multiple tenants (typically users), the active tenant is selected per session from the identity's memberships (queried from the Tenant service).

See [Authentication](authn.md) for how identity and tenant context are propagated.
