# Multi-Tenancy

## Overview

The platform supports multiple tenants. A **tenant** is an isolated organizational unit that owns all resources (agents, MCP servers, workspaces, models, threads, chats, etc.). Resources are scoped to a tenant — they are not visible or accessible across tenant boundaries.

## Tenant Model

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique tenant identifier |
| `name` | string | Display name |
| `created_at` | timestamp | Creation time |

## Tenant Service

The Tenant service is a **control plane** service.

### Responsibilities

| Concern | Description |
|---------|-------------|
| **Tenant CRUD** | Create, read, update, delete tenants |
| **List accessible tenants** | Return tenants an identity can access. Queries [Authorization](authz.md) for the list of tenant IDs, then enriches with tenant details from its own database |

### Data Store

PostgreSQL — `tenants` table.

## Tenant Access

Tenant access is managed through [Authorization](authz.md) (OpenFGA relationship tuples). See [Authorization — Tenant Permissions](authz.md#tenant-permissions) for the permission model.

### Per-Request Validation

The [Gateway](gateway.md) receives a `tenant_id` in request headers and validates access via the Authorization service (`Check`).

### Tenant Listing

When the UI needs to display a tenant switcher, it calls the Tenants service `ListTenants`. The Tenants service queries Authorization (`ListObjects`) for accessible tenant IDs, then returns full tenant details from its own database.

## Identities and Tenants

Any identity — user, agent, channel, or runner — can have access to a tenant. What an identity can do within a tenant is determined by its [authorization relationships](authz.md), not by its type.

An identity can have access to multiple tenants. For users, the active tenant is selected per session by the client. For non-user identities (agents, channels, runners), the tenant is typically fixed at creation.

See [Identity](identity.md) for the identity registry and [Authentication](authn.md) for how tenant context is propagated.

## Resource Scoping

Every resource in the platform belongs to a tenant. The `tenant_id` is present on all resource tables and used in all queries as a mandatory filter.

| Service | Scoped Resources |
|---------|-----------------|
| Agents | Agents, Volumes, Volume Attachments, MCPs, Skills, Hooks, ENVs, InitScripts |
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

Every authenticated request carries a `tenant_id`. The [Gateway](gateway.md) receives it in request headers, validates access via [Authorization](authz.md), and propagates the validated `tenant_id` in gRPC metadata to downstream services.

See [Authentication](authn.md) for how identity and tenant context are propagated.
