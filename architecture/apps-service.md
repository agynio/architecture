# Apps Service

## Overview

The Apps Service manages apps and installations — the configuration entities that define [apps](apps.md), their profiles, their visibility, and their per-organization installations. It handles both control plane operations (app management, installation) and data plane operations (profile resolution, slug lookup on the Gateway request path).

## API

| Method | Description |
|--------|-------------|
| **CreateApp** | Create a new app. Creates the app record, registers an identity (type `app`) in [Identity](identity.md), and generates a service token. Requires ownership of the organization |
| **GetApp** | Get an app by ID |
| **GetAppBySlug** | Get an app by owning organization ID + slug |
| **ListApps** | List apps. Supports filtering by organization (own apps) and visibility (public apps) |
| **UpdateApp** | Update an app (name, description, icon, visibility) |
| **DeleteApp** | Delete an app. Revokes the app's OpenZiti identity. Fails if active installations exist |
| **GetAppProfile** | Get an app's display profile (name, icon, description). Used by [Chat](chat.md) to render app-originated messages |
| **InstallApp** | Install an app into an organization. Creates the installation record and writes authorization tuples. Requires org ownership and that the app's visibility allows it |
| **GetInstallation** | Get an installation by ID |
| **GetInstallationBySlug** | Get an installation by slug within an organization. Used by the [Gateway](gateway.md) for [app proxy](gateway.md#app-proxy) routing |
| **ListInstallations** | List installations. Supports filtering by organization and by app |
| **UpdateInstallation** | Update an installation (slug, configuration) |
| **UninstallApp** | Delete an installation. Removes authorization tuples |
| **GetInstallationConfiguration** | Get the configuration for an installation. Called by the app to retrieve its configuration for a specific installation |

## App Resource

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique app identifier |
| `organization_id` | string (UUID) | Owning organization |
| `slug` | string | Unique within the owning organization. Used in the app's public address (`{org-slug}/{app-slug}`) and as default installation slug |
| `name` | string | Display name (e.g., "Telegram Connector") |
| `description` | string | Human-readable description |
| `icon` | string | Icon URL or identifier for UI display |
| `visibility` | enum | `public`, `internal` |
| `identity_id` | string (UUID) | App's identity in the [Identity](identity.md) service |
| `service_token_hash` | string | SHA-256 hash of the service token. Used for enrollment |
| `created_at` | timestamp | Creation time |
| `updated_at` | timestamp | Last modification time |

## Installation Resource

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique installation identifier |
| `app_id` | string (UUID) | Reference to the app |
| `organization_id` | string (UUID) | The organization this installation belongs to |
| `slug` | string | Unique within the installing organization. Used in CLI commands and Gateway routing. Defaults to the app's slug |
| `configuration` | map (string → string) | App-specific configuration. Opaque to the platform |
| `created_at` | timestamp | Creation time |
| `updated_at` | timestamp | Last modification time |

## App Flow

```mermaid
sequenceDiagram
    participant Dev as App Developer
    participant AS as Apps Service
    participant I as Identity

    Dev->>AS: CreateApp(org_id, slug, name, visibility)
    AS->>I: RegisterIdentity(id, type: app)
    AS->>AS: Generate service token, store app record
    AS-->>Dev: App record + service token
```

1. App developer calls `CreateApp` within their organization.
2. Apps Service registers the app's identity in the [Identity](identity.md) service with `identity_type: app`.
3. Apps Service generates a long-lived service token, stores the app record, and returns the token.
4. The service token is provided to the app deployment.

## Installation Flow

```mermaid
sequenceDiagram
    participant Admin as Org Admin
    participant AS as Apps Service
    participant Auth as Authorization

    Admin->>AS: InstallApp(app_id, organization_id, slug, configuration)
    AS->>AS: Validate: app visibility allows installation
    AS->>AS: Validate: slug unique within org
    AS->>AS: Store installation record
    AS->>Auth: Write(identity:appIdentityId, can_create_thread, organization:orgId)
    AS->>Auth: Write(identity:appIdentityId, can_add_participant, organization:orgId)
    AS-->>Admin: Installation record
```

1. Org admin calls `InstallApp` with the app ID, target organization, slug, and configuration.
2. Apps Service validates that the app's visibility allows installation by this organization (`public` — any org; `internal` — owning org only).
3. Apps Service validates slug uniqueness within the target organization.
4. Apps Service stores the installation record.
5. Apps Service writes authorization tuples granting the app's identity permissions within the organization (create threads, add participants).
6. Returns the installation record.

## Uninstall Flow

```mermaid
sequenceDiagram
    participant Admin as Org Admin
    participant AS as Apps Service
    participant Auth as Authorization

    Admin->>AS: UninstallApp(installation_id)
    AS->>Auth: Delete(identity:appIdentityId, can_create_thread, organization:orgId)
    AS->>Auth: Delete(identity:appIdentityId, can_add_participant, organization:orgId)
    AS->>AS: Delete installation record
    AS-->>Admin: OK
```

When an installation is removed, the authorization tuples are deleted. The app loses access to the organization. Existing threads where the app is a participant remain — the app can no longer create new threads or add participants, but its historical messages are preserved.

## Enrollment

When the app starts, it presents the service token to the platform enrollment endpoint. The platform validates the token, creates an OpenZiti identity via [Ziti Management](openziti.md), enrolls it, and returns the enrolled identity (certificate + key) to the app. This follows the same flow as [external runner enrollment](openziti.md#runner-provisioning).

After enrollment, the app can:

- **Bind** its OpenZiti service — Gateway can now route commands to it.
- **Dial** the Gateway — the app can call platform APIs.

The service token is long-lived and can be reused. If the app restarts, it re-enrolls with the same token and receives a new OpenZiti identity. The previous identity is cleaned up by Ziti Management lease GC.

## Profile Resolution

When [Chat](chat.md) encounters a message with `sender_id` of type `app` (resolved via [Identity](identity.md)), it calls `GetAppProfile` to fetch the display profile (name, icon).

## Data Store

PostgreSQL. Apps Service owns its database — `apps` and `app_installations` tables.

## Classification

| Aspect | Detail |
|--------|--------|
| **Plane** | Mixed — control (definition, installation) + data (profile/slug resolution) |
| **API** | gRPC (internal) + Gateway (external via ConnectRPC) |
| **State** | PostgreSQL |
