# Multi-Tenancy

## Overview

The platform supports multiple tenants. A **tenant** is an isolated organizational unit that owns all resources (agents, MCP servers, workspaces, models, threads, chats, etc.). Resources are scoped to a tenant — they are not visible or accessible across tenant boundaries.

## Tenant Model

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique tenant identifier |
| `name` | string | Display name |
| `created_at` | timestamp | Creation time |

## Users and Tenants

Users authenticate via a system-wide OIDC provider (see [Authentication](authn.md)). After authenticating, a user can:

- **Create a tenant** — becomes the owner.
- **Be granted access** to an existing tenant by the tenant owner.
- **Belong to multiple tenants** — selects the active tenant in the UI.

The user-tenant relationship is many-to-many. Tenant membership is managed within the platform, not in the IdP.

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

## Identity and Tenant Association

Every authenticated identity (user, agent, channel, runner) is associated with a tenant. For non-user identities (agents, channels, runners), the tenant is fixed — determined at identity creation. For users, the active tenant is selected per session from the user's tenant memberships. The tenant is resolved during authentication and propagated in request context to all downstream services. See [Authentication](authn.md).
