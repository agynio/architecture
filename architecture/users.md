# Users

## Overview

The Users service manages user identity records and user profiles. It is the platform's source of truth for user existence and user-facing metadata (name, nickname, photo).

The Users service does not manage credentials — authentication is handled by an external OIDC provider via the [Gateway](gateway.md). It does not manage tenant membership — that belongs to the [Tenant](tenancy.md) service. It does not manage non-user identities — agents, channels, and runners are owned by the services that create them ([Teams](teams.md), [Channels](channels.md), etc.).

All user records are system-wide (not scoped to a tenant). A user exists independently of any tenant and may belong to zero or more tenants.

## Responsibilities

| Concern | Description |
|---------|-------------|
| **User provisioning** | Create a user record on first OIDC login. Maps the IdP subject to a platform `identity_id` |
| **User profile** | Store and serve user profile data (name, nickname, photo URL) |
| **User lookup** | Resolve a user by `identity_id` or by OIDC subject |
| **Batch profile resolution** | Return profiles for a list of identity IDs. Used by Chat, UI, and other consumers that display user information |

## User Model

| Field | Type | Description |
|-------|------|-------------|
| `identity_id` | string (UUID) | Platform identity identifier. Used as `identity_id` in request context and authorization |
| `oidc_subject` | string | Subject claim from the OIDC ID token. Unique. Used to match returning users |
| `name` | string | Display name |
| `nickname` | string | Short name or handle |
| `photo_url` | string | Profile photo URL |
| `created_at` | timestamp | When the user was first provisioned |
| `updated_at` | timestamp | Last profile update |

## Provisioning Flow

When a user authenticates via OIDC for the first time, the Gateway calls the Users service to provision a user record:

```mermaid
sequenceDiagram
    participant U as User (Browser)
    participant GW as Gateway
    participant IdP as External IdP
    participant US as Users

    U->>GW: Request (no session)
    GW->>U: Redirect to IdP
    U->>IdP: Authenticate
    IdP->>U: Authorization code
    U->>GW: Authorization code
    GW->>IdP: Exchange code for tokens
    IdP->>GW: ID token + access token
    GW->>US: ResolveOrCreateUser(oidc_subject, initial profile from ID token)
    US-->>GW: identity_id
    GW->>GW: Establish session with identity_id
    GW->>U: Session established
```

On subsequent logins, `ResolveOrCreateUser` returns the existing `identity_id` without creating a new record.

Initial profile fields (name, photo) are populated from the OIDC ID token claims at provisioning time. The user can update their profile later.

## Consumers

| Consumer | Usage |
|----------|-------|
| **Gateway** | Resolve OIDC subject → `identity_id` on every user authentication |
| **Chat** | Resolve user profiles for message display (sender name, photo) |
| **UI** | Display user profile, profile editing |
| **Tenant service** | Display member information in tenant membership lists |

## Identity Model Context

The platform has four identity types: user, agent, channel, runner. All identity types are equal in the [authorization model](authz.md) — they are represented as `identity:<identity_id>` in OpenFGA. What an identity can do is determined by its relationships (tenant membership, resource access), not by its type.

Each identity type has its own provisioning path and profile shape, managed by different services:

| Identity Type | Provisioned By | Profile Owner | Authentication |
|---------------|---------------|---------------|----------------|
| **User** | Users service (this service) | Users service | OIDC via Gateway |
| **Agent** | [Teams](teams.md) | [Teams](teams.md) | OpenZiti via [Ziti Management](openziti.md) |
| **Channel** | [Channels](channels.md) | [Channels](channels.md) | OpenZiti via [Ziti Management](openziti.md) |
| **Runner** | Runner enrollment flow | [Ziti Management](openziti.md) | OpenZiti via [Ziti Management](openziti.md) |

The `identity_type` field in request context indicates the authentication mechanism and profile source, not authorization scope. See [Authentication](authn.md).

## Data Store

PostgreSQL. The `users` table is system-wide (not partitioned by tenant).

## Classification

The Users service is a **data plane** service — it is on the hot path for user authentication (Gateway calls it on every OIDC login) and profile resolution (Chat and UI call it to display user information).
