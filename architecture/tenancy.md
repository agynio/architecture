# Multi-Tenancy

## Overview

The platform supports multiple tenants. A **tenant** is an isolated organizational unit that owns all resources (agents, MCP servers, workspaces, models, threads, chats, etc.). Resources are scoped to a tenant — they are not visible or accessible across tenant boundaries.

The **Tenant service** manages tenant lifecycle. It is the source of truth for which tenants exist.

## Tenant Model

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique tenant identifier |
| `name` | string | Display name |
| `created_at` | timestamp | Creation time |

## Tenant Service

The Tenant service is a **control plane** service. It manages tenant resources — create, read, update, delete.

### Responsibilities

| Concern | Description |
|---------|-------------|
| **Tenant CRUD** | Create, read, update, delete tenants |

The Tenant service does not manage membership or access control. Which identities can access a tenant is determined by relationship tuples in [OpenFGA](authz.md) — managed through the [Authorization](authz.md) service.

### Data Store

PostgreSQL — `tenants` table.

## Tenant Access

Access to tenants is managed through the [Authorization](authz.md) service (OpenFGA). When an identity creates a tenant, the caller writes an ownership relationship tuple (e.g., `identity:<id>` is `owner` of `tenant:<tenantId>`) to the Authorization service. Granting other identities access to a tenant is also an authorization relationship write.

The [Gateway](gateway.md) resolves which tenants an identity can access by querying the Authorization service (`ListObjects(identity:<id>, member, tenant)`). The Tenant service is not involved in this resolution.

### Tenant Permissions

Defined in the [authorization model](authz.md):

| Permission | Capabilities |
|------------|-------------|
| **owner** | Full access. Manage tenant settings, membership, all resources. Delete tenant |
| **member** | Chat. View tracing. View resources (read-only) |

`owner` implies `member`. Any identity type can hold tenant permissions. For example, an agent that creates a tenant becomes its owner. See [Authorization](authz.md).

## Identities and Tenants

Any identity — user, agent, channel, or runner — can have access to a tenant. The identity type does not restrict tenant operations. What an identity can do within a tenant is determined by its relationships in the [authorization model](authz.md), not by its type.

An identity can have access to multiple tenants. For users, the active tenant is selected per session. For non-user identities (agents, channels, runners), the tenant is typically fixed — determined at identity creation.

See [Identity](identity.md) for the identity registry and [Authentication](authn.md) for how tenant context is propagated in requests.

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

For identities with a single tenant (typically agents, channels, runners), the tenant is fixed — determined at identity creation. For identities with multiple tenants (typically users), the active tenant is selected per session. The Gateway resolves available tenants by querying the [Authorization](authz.md) service.

See [Authentication](authn.md) for how identity and tenant context are propagated.
