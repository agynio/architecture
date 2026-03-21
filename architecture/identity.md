# Identity

## Overview

The Identity service is the platform's central registry of all identities. It maps every `identity_id` to its `identity_type`. Every service that creates an identity registers it here first.

## Why

The platform has four identity types (user, agent, channel, runner), each provisioned and profiled by a different service. Services like [Threads](threads.md) store only opaque identity UUIDs. Consumers that need to display identity information (e.g., [Chat](chat.md) showing sender name and photo) query the Identity service to determine the type, then fetch the profile from the appropriate source.

## Identity Model

| Field | Type | Description |
|-------|------|-------------|
| `identity_id` | string (UUID) | Unique identity identifier. Primary key |
| `identity_type` | enum | `user`, `agent`, `channel`, `runner` |
| `created_at` | timestamp | Registration time |

## Interface

| Method | Description |
|--------|-------------|
| **RegisterIdentity** | Register a new identity (id + type). Called by the service that creates the identity |
| **GetIdentityType** | Return the type for a single identity ID |
| **BatchGetIdentityTypes** | Return types for a list of identity IDs |

## Registration

Every service that creates an identity registers it here:

| Identity Type | Registering Service | When |
|---------------|-------------------|------|
| **User** | [Users](users.md) | On first OIDC login (user provisioning) |
| **Agent** | [Agents](agents-service.md) | On agent resource creation |
| **Channel** | [Channels](channels.md) | On channel creation |
| **Runner** | Runner enrollment flow | On runner enrollment |

The registering service generates the `identity_id` (UUID) and calls `RegisterIdentity` before storing the identity in its own database.

## Consumers

| Consumer | Usage |
|----------|-------|
| **Chat** | Resolve `sender_id` → type, then route to Users or Agents for profile |
| **UI** | Resolve identity types for display in membership lists, thread participants |

## Data Store

PostgreSQL. A single `identities` table. System-wide — not scoped to an organization. See [Organizations](organizations.md).

## Classification

**Data plane** — on the hot path for profile resolution.
