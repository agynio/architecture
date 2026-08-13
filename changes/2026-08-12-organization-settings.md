# Organization Settings and Deletion

## Target

- [Console — Settings](../product/console/console.md#settings) (new)
- [Console — Organization Sections](../product/console/console.md#organization-sections)
- [Console — Destructive Actions](../product/console/console.md#destructive-actions)
- [Console — Organizations (Cluster Admin)](../product/console/console.md#organizations-cluster-admin)
- [Organizations — Deletion](../architecture/organizations.md#deletion) (new)
- [Organizations — Organization Model](../architecture/organizations.md#organization-model)
- [Organizations — Slug](../architecture/organizations.md#slug)
- [Organizations — Interface](../architecture/organizations.md#interface)
- [Organizations — Teardown Order](../architecture/organizations.md#teardown-order) (new)
- [Authorization — Organization Permissions](../architecture/authz.md#organization-permissions)
- [Authorization — Organizations Service](../architecture/authz.md#organizations-service)
- [Console — Gateway API Surface](../architecture/console.md#gateway-api-surface)

## Delta

### An organization cannot be edited after it is created

The create form in the context switcher is the only place a name or a [slug](../architecture/organizations.md#slug) is ever set. The Console has no page that calls `UpdateOrganization` or `DeleteOrganization` — the generated clients are the only references to either method in `console-app`. An owner who typos a slug, or whose team renames itself, has an API call as their only remedy.

The organization's [sandbox defaults](../architecture/organizations.md#sandbox-settings) are in the same position, and have been since they were introduced: `UpdateOrganization` accepts all three, validates them against the platform bounds, and no surface sends them. This change gives them their first home rather than a second one.

### `DeleteOrganization` is unauthorized

The method parses the ID and deletes the row. It performs no `Check` — the only method in the Organizations service that does not — and the [Gateway](../architecture/gateway.md) forwards it without one either. Any authenticated caller who knows an organization's UUID can delete it.

### Deleting an organization deletes a row

`DELETE FROM organizations WHERE id = $1`. The `memberships` rows follow through an FK cascade. Nothing else does:

- **Every org-scoped record in every other service survives**, holding an `organization_id` that resolves to nothing: agents, environments, sandboxes, agent instances, images, LLM providers and models, secret providers and secrets, chats, apps and their installations, networks and private resources, egress rules, groups, org-scoped runners, threads and messages.
- **The OpenFGA tuples survive.** No tuple is deleted — the FK cascade is a database mechanism and the service writes nothing to Authorization. `identity:<owner>, owner, organization:<id>` outlives the organization, and the orphaned records above are still authorized by it. The former owner keeps full access to every one of them through the API and the CLI, having lost only the Console listing that would have shown them.
- **The workloads keep running.** Nothing stops them, and the Runners records that would let the [Orchestrator](../architecture/agents-orchestrator.md) recognize them as orphans are still there.

There is no `state` field and no teardown of any kind. No service exposes `DeleteOrganizationResources`, and none of them would be called if it did.

### `can_manage_organization` does not exist

The specs already disagree about who may update or delete an organization: [Console — Gateway API Surface](../architecture/console.md#gateway-api-surface) said "org owner or cluster admin", the product spec granted cluster admins rename and delete from Cluster Administration, and the authorization model required `owner` — which a cluster admin does not hold, on any organization. The target resolves this the way [`can_manage_members`](../architecture/authz.md#organization-permissions) was resolved: a computed relation including `admin from cluster`, so the Cluster Administration surface that claims these actions can actually perform them.

## Acceptance Signal

- **Settings** appears last in the Console's Organization sidebar group, for organization owners and cluster admins.
- Name and slug are editable there. A slug that is already taken is rejected with the server's message and the field keeps its value; a valid slug applies only after the consequences dialog is confirmed, never optimistically.
- The three sandbox defaults are editable there, are validated against the platform bounds, and do not change any sandbox that already exists.
- `UpdateOrganization` and `DeleteOrganization` both check `can_manage_organization`, and `can_manage_organization` resolves for `owner` and for `admin from cluster`. An authenticated non-member is refused on both.
- Deleting an organization requires typing its name.
- On confirmation the organization moves to `deleting`, its `member` and `owner` tuples are gone, and it has left the confirming user's context switcher. A second `DeleteOrganization` on it succeeds without error; every other write to it is refused. Its memberships are still on record.
- Every org-scoped service exposes `DeleteOrganizationResources(organization_id)`, callable by Organizations alone, and Organizations walks them in [teardown order](../architecture/organizations.md#teardown-order). Each step is idempotent — calling it twice succeeds twice.
- A step that fails is retried and the cascade does not advance past it. The organization stays `deleting` at that step rather than losing its record.
- Workloads and provisioned volumes belonging to the organization are gone after step 2, stopped and deprovisioned by the Orchestrator's existing orphan reconciliation rather than by a new teardown call.
- The organization row, its memberships, and its slug survive every step and are removed together at the end. Creating a new organization with that slug is rejected as taken until then, and succeeds after.
- No OpenFGA tuple on `organization:<id>` remains once the organization is gone.
- Cluster Administration → Organizations lists `deleting` organizations with the step each has reached.

## Notes

The teardown order is a configured list, and a service that owns org-scoped records but is absent from it is never called. That is the maintenance cost of deleting the organization only once its resources are genuinely gone. It also buys the slug hold: releasing the name at the request instead would hand it to a new organization while live exposures still answer on `<entity>.<old-slug>.agyn`.

Organizations gains an outbound call to every org-scoped service, which inverts the usual direction — services depend on Organizations for org existence, not the reverse. A cascade driven from the owner of the thing being deleted is the price of ordering, and ordering is what lets each service use its existing delete paths instead of a second one that skips its own reference checks.

Two things are deliberately not deleted and are worth stating rather than discovering. [Metering](../architecture/metering.md) and [tracing](../architecture/tracing.md) data age out under their own retention instead of being deleted on request, because deleting on request rewrites billing history. File objects stay in object storage because the [Files service](../architecture/media.md) has no deletion path at all — not an exception made for organization deletion, but a gap that predates it and that this change does not close.

Step 3 reaches outside the organization: deleting a publisher uninstalls its public apps from every organization that installed them. There is no notice to those organizations, which is a product question this change does not answer.

`agyn` has no organization commands, so the Console remains the only surface for any of this. The user-facing docs in `agyn/platform/docs/administer/organizations.md` already describe editing the name and a "Danger zone" on the **Overview** page — neither of which exists. Both belong to **Settings**, and the delete flow described there ("the org disappears from the context switcher immediately") needs the `deleting` state behind it.
