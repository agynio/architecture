# Identity

## Overview

The Identity service is the platform's central registry of all identities. It maps every `identity_id` to its `identity_type`. Every service that creates an identity registers it here first.

## Why

The platform has five identity types (user, agent, agent_instance, runner, app), each provisioned and profiled by a different service. Services like [Threads](threads.md) store only opaque identity UUIDs. Consumers that need to display identity information (e.g., [Chat](chat.md) showing sender name and photo) query the Identity service to determine the type, then fetch the profile from the appropriate source.

## Identity Model

| Field | Type | Description |
|-------|------|-------------|
| `identity_id` | string (UUID) | Unique identity identifier. Primary key |
| `identity_type` | enum | `user`, `agent` (class), `agent_instance`, `runner`, `app`, `platform` |
| `created_at` | timestamp | Registration time |

`agent` identifies an agent **class** (the configuration entity in [Agents Service](agents-service.md)); `agent_instance` identifies a specific [instance](agent-instances.md) of a class. Both may appear as `sender_id` on messages; only `agent_instance` appears as a thread participant (classes are rewritten to instances on add — see [Threads — Class-on-Add Rewrite](threads.md#class-on-add-rewrite)).

## Nickname Index

Nicknames are stored in a separate `org_nicknames` table, independent of the core identity record. This allows users to have different nicknames in different organizations.

| Field | Type | Description |
|-------|------|-------------|
| `org_id` | string (UUID) | Organization scope. Always set — there are no global nicknames |
| `identity_id` | string (UUID) | The identity this nickname belongs to (class or instance) |
| `installation_id` | string (UUID), nullable | Set for app installations; null for users, agents, and agent instances |
| `nickname` | string | The class handle. Pattern: `^[a-z0-9_-]+$`, max 32 chars |
| `instance_suffix` | string, nullable | Set only for `agent_instance` identities. Combined with `nickname` to form `@nickname#instance_suffix`. Either a system-generated stem (opaque, e.g., `7a2f`) or a user-chosen label. Pattern: `^[a-z0-9_-]+$`, max 32 chars |

Constraints: `UNIQUE(org_id, nickname, instance_suffix)` — one full handle per org (with `instance_suffix` treated as an empty string when NULL). For users, agents (classes), and agent instances: `UNIQUE(org_id, identity_id)` — one handle per identity per org. For app installations: `UNIQUE(org_id, installation_id)`.

`ResolveNickname(handle, org_id)` accepts either `@nickname` or `@nickname#instance_suffix`. It returns `identity_id`, `identity_type`, and `installation_id`. When the handle includes `#suffix`, the returned identity is an `agent_instance`; without `#`, it is a `user`, `agent` (class), or `app`. Callers use `identity_type` to route profile fetches, and `installation_id` (when set) to identify which app installation configuration applies in a given thread context.

All identity types use the same table. For users, the [Organizations](organizations.md#default-nickname-on-activation) service seeds a default nickname from the user's cluster-wide [`username`](users.md#username) when their membership becomes active; on conflict the seeding is skipped and the user picks one later. Agent classes register their nickname at creation time. Agent instances register at instance-create time — the nickname is the class's nickname; the `instance_suffix` is either supplied by the caller (as a `label`) or system-generated. App installations register a nickname per installation, defaulting to the app's slug. `@mention` resolution is always org-scoped — `@alice` in org A and `@alice` in org B are independent entries that may refer to different identities.

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
| **Agent** (class) | [Agents](agents-service.md) | On agent resource creation |
| **Agent Instance** | [Agents](agents-service.md) | On instance creation (`CreateInstance`, lazy via `Threads.CreateThread`/`AddParticipant`, or explicit) |
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

## Related Concepts

**Groups are not part of the Identity service.** Groups are organization-scoped collections of identities used for bulk permission grants. Their CRUD, membership tracking, OpenFGA tuple writes, and OpenZiti role-attribute sync live in the dedicated [Groups service](groups-service.md). The Identity service is kept narrowly focused on identity registration and the cross-type type lookup that consumers like [Chat](chat.md) need on the hot path; Groups grows its own surface area (SCIM, sync workers, source-of-truth tracking) that does not belong here.

## Data Store

PostgreSQL. System-wide — not scoped to an organization. See [Organizations](organizations.md).

## Classification

**Data plane** — on the hot path for profile resolution.
