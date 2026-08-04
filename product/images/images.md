# Images

## Purpose

Every container image the platform runs is named today as a free-form registry URL typed into the resource that uses it — `image` and `init_image` on an [agent](../../architecture/resource-definitions.md#agent), `image` on an [environment](../environments/environments.md) and on an [MCP](../../architecture/resource-definitions.md#mcp). That asks the person configuring an agent to know what a registry is, which repository holds the right artifact, and which of its tags pairs with the platform they are running. Most of them do not, and nothing in the platform helps: a URL is accepted whether or not it resolves, and its tags are invisible until a workload fails to start.

The **image catalog** replaces those URLs with a resource. An organization registers an image once — where it lives and how to authenticate — and everything downstream selects it by name from a list. Versions are not registered at all: the platform reads them from the upstream repository and keeps them current, so the catalog always shows what actually exists rather than what somebody remembered to write down.

The path is always registry → environment → agent or sandbox. No resource anyone configures holds a registry URL; the platform's own containers are pinned by its chart and are not a configuration surface.

## User Stories

- As an organization owner, I want to register an image once, so everyone in my organization selects it by name instead of copying a URL between resources.
- As someone creating an environment, I want to pick an image and a version from a list, so I never have to know what a registry is.
- As someone picking a version, I want the newest sensible one already selected, so the common case needs no decision at all.
- As an organization owner, I want to publish an image other organizations can use, so a shared base environment is defined once on the platform.
- As a platform operator, I want the platform's own images present the moment the platform is installed, with nothing to register by hand.
- As a security-conscious operator, I want registry credentials to stay inside the platform and never reach the clusters my workloads run on.

## Concepts

| Term | Definition |
|---|---|
| **Image** | An org-scoped record naming an upstream repository, the credential to read it, and what the image is for. The only thing anyone authors. |
| **Version** | One tag in that repository, **discovered** by the platform rather than registered. Carries the tag, when it was pushed, and its description. |
| **Type** | Which slot in a workload the image is built for: workspace, agent runtime, or MCP. |
| **Visibility** | Whether the image is usable by every organization on the platform or only the one that owns it. |

## Registering an Image

An image is a small record, and registering one is a single step:

| Field | Meaning |
|---|---|
| `name` | Unique within the organization. What every picker shows |
| `type` | `workspace`, `agent_runtime`, or `mcp` — see [Types](#types) |
| `repository` | The upstream repository (e.g. `ghcr.io/agynio/devcontainer-go`) |
| `username` / `password` | Credential for reading the repository. Optional for public repositories |
| `visibility` | `public` or `internal` — see [Visibility](#visibility) |
| `description` | Free text, shown beside the name |
| `tag_filter` | Optional pattern limiting which tags appear in pickers |

Nothing else is authored. There is no second step to register a version, no metadata to fill in per release, and no field anyone has to remember to update when the upstream repository moves on.

### Types

The type says which slot the image is built to occupy. It is what makes a picker show three devcontainers instead of every image in the organization.

| Type | Slot | Contents |
|---|---|---|
| `workspace` | The workload's main container | A devcontainer — the tools, runtimes and dependencies work happens in |
| `agent_runtime` | An init container | An agent CLI binary. See [Agent Init Container](../../architecture/agent-init.md) |
| `mcp` | A sidecar container | An [MCP](../../architecture/resource-definitions.md#mcp) server, with its command built into the image |

An MCP may run either an `mcp` image or a `workspace` image — a purpose-built server and a devcontainer are both legitimate ways to host one. The other two slots take only their own type.

The type does not say *which* agent CLI an `agent_runtime` image provides. That is settled inside the image, which carries the same self-describing configuration it does today, and is conveyed to a human by the name and description whoever registered it chose.

### Visibility

| Visibility | Who can use the image |
|---|---|
| `internal` | Members of the owning organization |
| `public` | Every organization on the platform |

The same two values, with the same meanings, that [apps](../../architecture/apps.md#visibility) already use. An image visible to an organization is usable by it — there is no separate grant, and no per-user or per-group sharing. Restricting *which* images an organization may build environments from is a governance question rather than a visibility one, and is deliberately left out; see [Future](#future).

A public image must be readable with the credential its owner registered, which the platform holds and never discloses. Consuming organizations supply nothing.

## Versions Are Discovered

Versions are read from the upstream repository, not written into the platform. The catalog polls each image's repository on an interval and refreshes on demand whenever someone opens a picker or an image's page, so a tag pushed upstream shows up without anyone touching the platform.

Each discovered version carries its tag, its push time, and the description the image declares about itself. That is enough to sort a list, label a row, and pick from it.

This is the whole reason the record is as small as it is. Registering a version by hand would mean pushing an image and then telling the platform about it — two steps to publish one artifact, and a catalog that silently drifts out of date the first time somebody skips the second one.

Two consequences follow, and both are accepted deliberately:

- **A tag is followed as written.** Repointing a tag upstream changes what runs on the next workload start. Nothing pins a digest, so nothing detects the change. See [Future](#future).
- **Browsing needs the upstream to be reachable.** In an air-gapped installation, images are registered against an internal mirror, which the platform polls like any other repository.

A tag that disappears upstream is marked gone rather than deleted. Environments still naming it are flagged unschedulable — the same treatment a [flavor](../environments/environments.md#placement) removed from a runner's catalog gets, and for the same reason: the reference is late-bound, so a missing target is a condition to surface, not a broken record to repair.

## Selecting a Version

A repository holds far more tags than anyone should be shown. The platform's own convention publishes `sha-<short>`, a semver, and `latest` into one repository, so even a well-behaved image arrives with three families of tag where only one is ever the right answer.

So the picker is opinionated:

- Tags that parse as semver are the list, newest first. Everything else sits behind **show all tags**.
- The newest semver tag is **preselected**, so creating an environment takes no version decision at all.
- Where a repository does not use semver, the image's `tag_filter` narrows the list, and push time orders it.
- Each row shows its push time and description, so tags are not bare strings.
- A tag can be typed directly for anyone who knows exactly what they want. It is checked against the repository before it is accepted.

There is no `latest` in the platform's own vocabulary. A tag by that name upstream is shown like any other, with no special standing — an alias the platform re-resolved on its own would make "which version is this environment running" unanswerable.

## How Images Are Pulled

No workload pulls a catalog image from its upstream registry. Every reference to one is rewritten to the platform's [image proxy](../../architecture/image-proxy.md), which authenticates upstream using the credential on the image record and streams the result back. The platform's own containers — the two binary init images and the Ziti sidecar — are pulled directly from a public registry instead, since the proxy cannot serve the components a workload needs before the proxy itself is reachable.

This is what keeps registry credentials inside the platform. Handing them to a runner means writing an organization's registry password into the cluster its workloads run in, where anyone able to read a secret there can take it — and for a public image consumed by another organization, it would mean handing one organization's credential to another's cluster outright.

Each workload is issued its own short-lived credential for the proxy, valid while that workload runs and revoked when it stops. The proxy decides, per request, whether that workload may pull that image.

It follows that the catalog is the only supply route a *user* has into a workload: nothing anyone configures can name an artifact the catalog does not hold, and the credentials to reach anything else are not present.

## Lifecycle

| Event | Effect |
|---|---|
| Image registered | Versions appear on the first discovery pass, usually within seconds |
| Tag pushed upstream | Appears on the next poll, or immediately when a picker is opened |
| Tag repointed upstream | Silent. Workloads started afterwards run the new contents |
| Tag removed upstream | Marked gone. Environments naming it are flagged unschedulable |
| Credential changed | Applies to the next discovery pass and the next pull |
| Visibility narrowed to `internal` | Other organizations' environments naming the image are flagged unschedulable |
| Image deleted | Allowed. Environments naming it are flagged unschedulable |
| Repository unreachable | Last known versions are served; the image is flagged stale |

Deleting an image is not blocked by references. Blocking it would mean an organization discovering its image is in use by another organization it cannot see, and the platform already has a shape for a reference whose target went away — flag it, do not repair it.

## Constraints

- An image belongs to exactly one organization. Consuming a public image creates no record in the consuming organization.
- Versions cannot be created, edited, or deleted through the platform. The upstream repository is the only source.
- Digests are not pinned. A reference names a tag, and the tag is resolved at each workload start.
- An `agent_runtime` image's compatibility with the platform is not modelled. Because [`agynd` ships with the platform](../../architecture/agent-init.md) rather than inside the image, a mismatch can only be an agent CLI the platform's adapters do not speak, which fails at startup with a clear error rather than silently.
- Architecture is not modelled. An image that cannot run on an environment's runner fails at workload start.
- Registering an image requires organization ownership. Using one requires only that it is visible.

## Future

- **Digest pinning** — recording the resolved digest on the reference, closing the repointed-tag gap and making "what actually ran" answerable.
- **Configuration schemas** — an `agent_runtime` image declaring the shape of an agent's `configuration`, so the Console builds a form instead of showing a JSON editor. The image already describes itself to `agynd`; this is the natural place for it.
- **Allow-lists** — organization owners restricting which images environments may be built from, which is the real form of the control `private` visibility would only have approximated.
- **Per-user and per-group sharing**, if a case appears that visibility alone cannot express.
- **Non-container images** — VM images and similar, entering as new `type` values.

## Related Architecture

- [Images Service](../../architecture/images-service.md)
- [Image Proxy](../../architecture/image-proxy.md)
- [Resource Definitions — Image](../../architecture/resource-definitions.md#image)
- [Flavors and Environments](../environments/environments.md)
- [Sandboxes](../sandboxes/sandboxes.md)
- [Agent Init Container](../../architecture/agent-init.md)
- [Console](../console/console.md)
