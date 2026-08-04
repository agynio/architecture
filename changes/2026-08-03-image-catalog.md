# Image Catalog

## Target

- [Images](../product/images/images.md)
- [Images Service](../architecture/images-service.md)
- [Resource Definitions — Image, Image Version](../architecture/resource-definitions.md#image)
- [Console — Images](../product/console/console.md#images)

## Delta

### Images Service

The service does not exist. No repository, no chart, no database, no Gateway wiring.

- The `Image` resource does not exist: no CRUD, no name uniqueness within an organization, no immutability on `repository` and `type`, no readability check against the upstream repository at create time.
- The `ImageVersion` resource does not exist. Nothing polls upstream repositories, nothing refreshes on demand, nothing marks a vanished tag `gone` or flags an image stale when its repository is unreachable, and tag metadata is never read.
- `ResolveVersion` does not exist, so nothing can validate an image reference on write or resolve one for a pull.
- `RegisterPlatformImage` does not exist — see [Platform Self-Registration](2026-08-03-platform-self-registration.md).
- No `image.updated` notification, so Console lists cannot react to discovery.

### Secrets Service

Registry credentials are a distinct `ImagePullSecret` resource with its own CRUD and resolution endpoint. Image records reference an ordinary [Secret](../architecture/providers.md#secret) instead; the separate resource and its resolution path go. Its removal is driven by [Image Pull Proxy](2026-08-03-image-pull-proxy.md), which is what makes it unnecessary.

### Authorization

No visibility handling for images. `public` readability by any authenticated identity and `internal` readability by organization members do not exist, and the per-operation requirements in [Images Service — Authorization](../architecture/images-service.md#authorization) are unenforced.

### Console

- No Images section. The Runtime sidebar group has no entry for it, and the ordering that places it before Environments does not exist.
- No image list, detail, or registration form; no version list; no staleness or foreign-organization indication.
- No version picker. Nothing filters by the type a slot requires, sorts semver tags newest-first, preselects the newest, hides other tags behind **show all tags**, or accepts a typed tag.
- The Credentials group still carries an Image Pull Secrets section.

### Terraform Provider

No `agyn_image` resource. `agyn_image_pull_secret` and `agyn_image_pull_secret_attachment` still exist and express the shape being removed.

## Acceptance Signal

- An organization owner registers an image with a name, type, repository and credential, and its tags appear without anyone entering a version.
- Registering an image against an unreadable repository, or with a wrong credential, fails at registration rather than at workload start.
- A tag pushed upstream appears in the picker after a poll, and immediately when the image page is opened.
- A tag removed upstream is marked `gone` rather than disappearing, and an image whose registry is unreachable still lists its last known versions and shows when discovery last succeeded.
- A `public` image registered in one organization is selectable from another; an `internal` one is not.
- The version picker preselects the newest semver tag and hides `sha-` and `latest` tags behind **show all tags**.
- Deleting an image succeeds while environments reference it, and those environments are flagged unschedulable.

## Notes

- Versions are not authored, and there is no create, update, or delete API for them. Anything that would require one — per-version descriptions, curation, blessing a release — is out of scope by construction.
- Digests are not pinned in this change. A reference names a tag, so repointing a tag upstream silently changes what runs. Recorded as [Future](../product/images/images.md#future).
- `private` visibility and per-user or per-group sharing are deliberately excluded. Restricting *which* images an organization may build on is a governance control rather than a visibility one, and is left to a later allow-list.
- Configuration schemas on `agent_runtime` images are excluded. Agents keep a free-form `configuration` JSON field with no validation and no generated form, which is the state before this change.
- Depends on [Image Pull Proxy](2026-08-03-image-pull-proxy.md) for the credential model: this change stores a credential on the image, and that one is what stops it from ever reaching a workload cluster.
- [Environment and Runtime Unification](2026-08-03-environment-runtime-unification.md) is what makes anything reference the catalog.
