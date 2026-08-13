# Images Service

## Overview

The Images service owns the [image catalog](../product/images/images.md) — the `Image` records an organization authors, and the `Version` records the platform discovers by reading upstream repositories. It is the only service in the platform that stores a registry URL. The credential to read that registry is an ordinary [Secret](secrets.md) the registrar creates, which the image names by id.

It is a separate service rather than part of [Agents](agents-service.md) because images cross organization boundaries. Everything the Agents service owns is org-scoped by construction (see [Organizations — Resource Scoping](organizations.md#resource-scoping)), while a `public` image is usable by every organization on the platform, and the [Image Proxy](image-proxy.md) resolves images on behalf of workloads in organizations that do not own them.

This is a **control plane** service. It stores what an organization declared and what the platform observed; nothing reconciles toward it directly.

## Concepts

| Term | Definition |
|---|---|
| **Image** | An org-scoped record: upstream repository, credential reference, type, visibility. The only authored resource |
| **Version** | A tag observed in that repository. Written only by discovery; never by a caller |
| **Discovery** | The background and on-demand process that reads tags from an upstream repository and reconciles the stored `Version` rows against them |

## Classification

| Aspect | Detail |
|---|---|
| **Plane** | Control |
| **Language** | Go |
| **Repository** | `agynio/images` |
| **API** | gRPC (internal) + Gateway (external via ConnectRPC) |
| **State** | PostgreSQL — `images` and `image_versions` tables |
| **External dependencies** | [Authorization](authz.md) (permission checks), [Secrets](secrets.md) (validating and resolving the credential an image names), [Notifications](notifications.md) (Console reactivity), upstream container registries |

## Responsibilities

| Responsibility | Description |
|---|---|
| **Image CRUD** | Create, read, update, delete `Image` records. Enforce name uniqueness within the organization |
| **Discovery** | Poll each image's repository on an interval and on demand; insert new versions, mark vanished ones gone, record staleness when the repository is unreachable |
| **Resolution** | Resolve an image reference to an upstream repository and credential for the [Image Proxy](image-proxy.md), and validate references for the [Agents](agents-service.md) service |
| **Visibility enforcement** | Serve `internal` images only to the owning organization; serve `public` images to every organization |
| **Platform registration** | Accept image records from the platform's own [self-registration](operations/local-bundle.md) path without a user identity |

## API

### Image CRUD

| Method | Description |
|---|---|
| **CreateImage** | Create an image. Requires `owner` on the organization. Validates the repository is readable with the credential named by `secret_id` before storing, so a typo fails at create rather than at workload start |
| **GetImage** | Fetch an image. Visible per [Visibility](#visibility) |
| **ListImages** | List images available to an organization: its own, plus every `public` image on the platform. Supports filtering by `type`. Cursor pagination |
| **UpdateImage** | Update `name`, `description`, `username`, `secret_id`, `visibility`, or `tag_filter`. A changed credential is read against the repository before it is stored, as at create. An empty `secret_id` drops the credential, so an image can be moved to an anonymously readable repository. `repository` and `type` are immutable — both are statements about what the record *is*, and changing either silently redefines every environment referencing it |
| **DeleteImage** | Delete an image and its versions. Not blocked by references; see [Deletion](#deletion) |

### Versions

| Method | Description |
|---|---|
| **ListVersions** | List discovered versions for an image, newest first. Serves stored rows — never blocks on the upstream registry |
| **RefreshImage** | Trigger an immediate discovery pass. Called when a picker or image page opens, so a freshly pushed tag appears without waiting for the poll |
| **ResolveVersion** | Resolve `(image_id, tag)` to an upstream reference. Internal; used by [Agents](agents-service.md) to validate a reference on write and by the [Image Proxy](image-proxy.md) to serve a pull |
| **CountImagesReferencingSecret** | Report which images name a secret. Internal; called by [Secrets](secrets.md#referential-integrity) before a delete |

## Resource Shapes

### Image

See [Resource Definitions — Image](resource-definitions.md#image) for the canonical schema.

`repository` and `type` are immutable after creation. The credential is held **by reference**: the registrar creates a [Secret](secrets.md) and the image names it by `secret_id`, so a registry password can be rotated, shared between images, or backed by a remote provider like every other credential on the platform, and no password ever crosses this service's API. The username is stored plainly on the record.

The reference is validated on write with `ResolveSecretExists`, and the secret must belong to the image's organization. Without that check an owner could name another organization's secret and point the image at a registry they control, which would send them its value on the next discovery pass.

### Version

Discovered, never authored:

| Field | Type | Description |
|---|---|---|
| `image_id` | string (UUID) | Owning image |
| `tag` | string | The tag as it appears upstream |
| `pushed_at` | timestamp | When the tag was pushed, read from the upstream manifest |
| `description` | string | The image's own `org.opencontainers.image.description`, when it declares one |
| `state` | enum | `present` or `gone` — see [Discovery](#discovery) |
| `discovered_at` | timestamp | First time the platform observed this tag |

## Discovery

Each image's repository is polled on a configurable interval, and refreshed immediately on `RefreshImage`. A pass lists the repository's tags, inserts rows for tags not seen before, and marks rows `gone` for tags that are no longer listed.

Registry webhooks are not used. They are not portable — the registries the platform is expected to read from do not all offer them — so polling is the only mechanism that works uniformly, and an on-demand refresh covers the latency a poll interval would otherwise add.

**Tag metadata is read lazily.** Listing tags is one request; reading a tag's push time and description requires fetching its manifest, and a repository with hundreds of tags would make a full pass expensive. Metadata is resolved for tags that survive the image's `tag_filter`, and otherwise on first display.

**Vanished tags are marked, not deleted.** A `gone` version stays in the table so environments naming it can be reported as unschedulable rather than silently referring to nothing. A tag that reappears upstream returns to `present`.

**An unreachable repository does not empty the catalog.** Stored versions continue to be served and the image is flagged stale, with the time of the last successful pass. A registry outage degrades freshness, not the ability to start workloads from what is already known.

Discovery makes outbound HTTPS calls from the control plane to whatever registries organizations have registered. In restricted networks this is an egress requirement of the platform itself, distinct from workload egress.

## Visibility

| Visibility | Readable by |
|---|---|
| `internal` | Members of the owning organization |
| `public` | Any authenticated identity |

Visibility governs reading a record. There is no separate grant for *using* an image: an organization that can see an image can build environments on it. This matches the [Apps](apps.md#visibility) model, which uses the same two values with the same meanings.

## Deletion

Deleting an image is permitted regardless of references. Blocking would require the Images service to ask the Agents service which environments name a record, and would surface cross-organization usage to an owner who cannot see the organizations involved.

Instead, references are late-bound: an environment naming a deleted image, a `gone` tag, or an image whose visibility no longer reaches it becomes **unschedulable and is flagged**, exactly as an environment naming a [flavor](../product/environments/environments.md#placement) that a runner stopped reporting. The platform already treats a missing late-bound target as a condition to surface rather than a record to repair.

Deleting an image leaves the Secret it named alone. The Secret is the registrar's, and another image may well name it; deleting a secret that is still named is refused by [Secrets](secrets.md#referential-integrity) rather than prevented here.

## Authorization

| Operation | Requirement |
|---|---|
| `CreateImage`, `UpdateImage`, `DeleteImage` | `owner` on the image's organization |
| `GetImage`, `ListImages`, `ListVersions`, `RefreshImage` | `member` on the owning organization, or the image is `public` |
| `ResolveVersion`, `CountImagesReferencingSecret` | Internal callers only |

No OpenFGA type is introduced. Both visibility values resolve against existing organization relations, so images need no per-resource tuples and no share management.

## Client-Facing Updates

Discovery publishes `image.updated` to the owning organization's [Notifications](notifications.md) room when an image's version set changes or its staleness flips, so Console lists and pickers reflect new tags without a reload. Organizations consuming a `public` image do not receive its owner's notifications; their pickers refresh on open.

## Data Store

PostgreSQL — `images` and `image_versions`. `images` carries `organization_id`; versions inherit organization scope through the image. Unique on `(organization_id, name)` and on `(image_id, tag)`.

## Related Architecture

- [Images (product)](../product/images/images.md)
- [Image Proxy](image-proxy.md)
- [Resource Definitions — Image](resource-definitions.md#image)
- [Agents Service](agents-service.md)
- [Secrets](secrets.md)
- [Organizations](organizations.md)
