# Platform Resource Self-Registration

## Target

- [Platform Resource Provisioning](../architecture/operations/platform-provisioning.md)
- [Images Service — Internal Registration](../architecture/images-service.md#internal-registration)
- [Local Bundle](../architecture/operations/local-bundle.md)

## Delta

### Provisioning Mechanism

No general mechanism exists. The one precedent is [Authorization — Bootstrap](../architecture/authz.md#bootstrap), where Terraform writes the initial cluster admin straight into PostgreSQL — a one-shot path for one resource that bypasses the owning service entirely.

- There is no internal, non-Gateway provisioning surface and no service identity to call one with. Every creation path requires a signed-in user with organization ownership, which does not exist at install time.
- No create-if-absent semantics anywhere, so nothing can run on every upgrade without either failing on the second run or overwriting what an operator changed.
- Nothing runs provisioning as part of install or upgrade.

### Platform Organization

Nothing creates it. There is no organization for platform-shipped resources to live in, and no established answer for what owns one with no members — the platform has never needed an organization that no human belongs to.

### Images

`RegisterPlatformImage` does not exist, so the platform's own workspace and agent runtime images must be registered by hand in every installation before anything can run. This is the case that makes the mechanism necessary rather than desirable.

### Runners and Apps

The runner and apps shipped with the platform are provisioned by install-specific glue rather than by declaring themselves through the same path. Bringing them onto it is what makes this a mechanism rather than an images feature.

## Acceptance Signal

- A freshly installed platform has the platform organization and the platform's images present, with nothing registered by hand.
- Running an upgrade twice changes nothing the second time.
- An operator edits a provisioned image; the next upgrade leaves the edit in place.
- An operator deletes a provisioned image; the next upgrade recreates it.
- A cluster admin can administer the platform organization without being a member of it.
- No provisioning path writes to a service's database directly.

## Notes

- Provisioned resources carry no ownership marker and are not read-only. Create-if-absent is what makes re-running safe, rather than special-casing them in the API.
- The consequence is accepted: a release cannot correct metadata on a resource it provisioned earlier. For images this costs nothing, since a release publishes new tags rather than editing records.
- Sequencing is independent of the other image changes, but [Image Catalog](2026-08-03-image-catalog.md) has to land first for `RegisterPlatformImage` to have a resource to create.
