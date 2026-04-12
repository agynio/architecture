# Identity

## Overview

The Identity service is the platform's central registry of all identities. It maps every `identity_id` to its `identity_type`. Every service that creates an identity registers it here first.

## Why

The platform has four identity types (user, agent, runner, app), each provisioned and profiled by a different service. Services like [Threads](threads.md) store only opaque identity UUIDs. Consumers that need to display identity information (e.g., [Chat](chat.md) showing sender name and photo) query the Identity service to determine the type, then fetch the profile from the appropriate source.

## Identity Model

| Field | Type | Description |
|-------|------|-------------|
| `identity_id` | string (UUID) | Unique identity identifier. Primary key |
| `identity_type` | enum | `user`, `agent`, `runner`, `app` |
| `created_at` | timestamp | Registration time |

## Nickname Index

Nicknames are stored in a separate `org_nicknames` table, independent of the core identity record. This allows users to have different nicknames in different organizations.

| Field | Type | Description |
|-------|------|-------------|
| `org_id` | string (UUID) | Organization scope. Always set — there are no global nicknames |
| `identity_id` | string (UUID) | The identity this nickname belongs to |
| `installation_id` | string (UUID), nullable | Set for app installations; null for users and agents |
| `nickname` | string | The handle. Pattern: `^[a-z0-9_-]+$`, max 32 chars |

Constraints: `UNIQUE(org_id, nickname)` — one nickname per handle per org. For users and agents: `UNIQUE(org_id, identity_id)` — one nickname per identity per org. For app installations: `UNIQUE(org_id, installation_id)` — one nickname per installation per org (the same app identity may appear multiple times in the same org with different installation IDs).

`ResolveNickname` returns `identity_id`, `identity_type`, and `installation_id`. Callers use `identity_type` to route profile fetches, and `installation_id` (when set) to identify which app installation configuration applies in a given thread context.

All identity types use the same table. Users set a nickname when joining or first using an org. Agents register their nickname at creation time. App installations register a nickname per installation, defaulting to the app's slug. `@mention` resolution is always org-scoped — `@alice` in org A and `@alice` in org B are independent entries that may refer to different identities.

## Interface

| Method | Description |
|--------|-------------|
| **RegisterIdentity** | Register a new identity (id, type). Called by the service that creates the identity |
| **SetNickname** | Set or change the nickname for an identity within an org. Returns a conflict error if the nickname is already taken in that org |
| **RemoveNickname** | Remove a nickname entry for an identity within an org |
| **GetIdentityType** | Return the type for a single identity ID |
| **BatchGetIdentityTypes** | Return types for a list of identity IDs |
| **ResolveNickname** | Resolve `@nickname` within a given org. Returns `identity_id`, `identity_type`, and `installation_id` (null for users and agents) |

## Registration

Every service that creates an identity registers it here:

| Identity Type | Registering Service | When |
|---------------|-------------------|------|
| **User** | [Users](users.md) | On first OIDC login (user provisioning) |
| **Agent** | [Agents](agents-service.md) | On agent resource creation |
| **Runner** | [Runners](runners.md) | On runner registration |
| **App** | [Apps Service](apps-service.md) | On app registration |

The registering service generates the `identity_id` (UUID) and calls `RegisterIdentity` before storing the identity in its own database.

## Consumers

| Consumer | Usage |
|----------|-------|
| **Chat** | Resolve `sender_id` → type, then route to Users or Agents for profile |
| **UI** | Resolve identity types for display in membership lists, thread participants |
| **Threads** | Resolve `@nickname` to `identity_id` when adding participants |

## Access

Internal only — not exposed through the [Gateway](gateway.md). Owner services (Users, Agents, Apps) call the Identity service to manage nicknames on behalf of their resources. External clients interact with nicknames through those services, not through the Identity service directly.

## Data Store

PostgreSQL. System-wide — not scoped to an organization. See [Organizations](organizations.md).

## Classification

**Data plane** — on the hot path for profile resolution.
