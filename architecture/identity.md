# Identity

## Overview

The Identity service is the platform's central registry of all identities. It maps every `identity_id` to its `identity_type`. Every service that creates an identity registers it here first.

The Identity service does not store profiles, credentials, tenant relationships, or permissions. It answers one question: **given an identity ID, what type is it?**

## Why

The platform has four identity types (user, agent, channel, runner), each provisioned and profiled by a different service. Services like [Threads](threads.md) work with opaque identity UUIDs and do not track type. Consumers that need to display identity information (e.g., [Chat](chat.md) showing sender name and photo) need to know the type to route to the correct profile source.

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

Every service that creates an identity must register it in the Identity service before using it:

| Identity Type | Registering Service | When |
|---------------|-------------------|------|
| **User** | [Users](users.md) | On first OIDC login (user provisioning) |
| **Agent** | [Teams](teams.md) | On agent resource creation |
| **Channel** | [Channels](channels.md) | On channel creation |
| **Runner** | Runner enrollment flow | On runner enrollment |

The registering service generates the `identity_id` (UUID) and calls `RegisterIdentity` before storing the identity in its own database.

## Consumers

| Consumer | Usage |
|----------|-------|
| **Chat** | Resolve `sender_id` → type, then route to Users or Teams for profile |
| **UI** | Resolve identity types for display in membership lists, thread participants, etc. |

## Data Store

PostgreSQL. A single `identities` table. System-wide (not scoped to a tenant).

## Classification

The Identity service is a **data plane** service — it is on the hot path for profile resolution by consumers like Chat and UI.
