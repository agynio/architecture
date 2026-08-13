# A Registry Credential Is a Secret an Image Names

## Target

- [Images Service — Resource Shapes](../architecture/images-service.md#resource-shapes)
- [Images Service — API](../architecture/images-service.md#api)
- [Images Service — Deletion](../architecture/images-service.md#deletion)
- [Images Service — Authorization](../architecture/images-service.md#authorization)
- [Secrets — Referential Integrity](../architecture/secrets.md#referential-integrity)
- [Resource Definitions — Image](../architecture/resource-definitions.md#image)
- [Authorization — Images Service](../architecture/authz.md#images-service)
- [Images — What an Image Record Holds](../product/images/images.md)
- [Console — Images](../product/console/console.md)

## Delta

**A registry password is typed into the Console and posted to the Images service as a string.** The service turns it into a Secret behind the caller's back, which is where [Providers](../architecture/providers.md#registry-credential-setup) already said the credential lives — as a Secret *referenced by* an Image — but the API never got there. Three things follow from the gap: an organization cannot point an image at a secret it already holds, a Vault-backed credential is unreachable for registries alone, and the Secret the service creates on the caller's behalf appears on the Secrets tab as an ordinary row that anyone with `owner` can edit or delete, silently breaking discovery.

`CreateImage` and `UpdateImage` take `secret_id` instead of `password`, matching [Subscription](../architecture/providers.md#subscription) and [EgressRule](../architecture/resource-definitions.md#egress-rule), the two resources that already hold a credential this way.

The pieces to build:

- `secret_id` on `CreateImageRequest` and `UpdateImageRequest`; `password` deprecated on both and no longer implemented. An empty `secret_id` on update drops the credential, which is how an image moves to an anonymously readable repository.
- Validation on write through `ResolveSecretExists`, including that the secret's organization is the image's. Existence alone is not enough — see the note below.
- `CountImagesReferencingSecret`, and Secrets calling it in `DeleteSecret` alongside the egress-rule and subscription checks. Without it the credential an image reads with can be deleted out from under it.
- A secret picker in the Console's register dialog, with **add a new secret** in the same Select so choosing one never means leaving the form. The Console creates the Secret and hands the Images service the id.
- Credential editing on the image detail page. `UpdateImage` has always accepted a credential change; nothing in the Console ever sent one, so a rotated registry password meant deleting the image and registering it again.

**The resolution path does not change.** `ResolveVersion` and `ResolveImage` still hand the [Image Proxy](../architecture/image-proxy.md) a username and a password: it is the one caller that needs the value, it is internal, and what it does with it is unchanged.

**The Images service no longer writes Secrets.** It validates a reference and resolves a value, both on the internal path. The dependency is now read-only in the direction that matters.

## Acceptance Signal

- An owner registers an image against a private repository by choosing a secret the organization already holds, and discovery reads the repository with it.
- An owner registers one by typing a password into **add a new secret**: the secret appears on the Secrets tab under its own name, and the image names it.
- An image registered against a public repository names no secret, and neither the form nor the record carries an empty credential to explain.
- Naming a secret that belongs to another organization is refused, even for an owner of both.
- Naming a secret that does not exist is refused at registration rather than at the first discovery pass.
- Rotating a registry password is done once on the Secret; every image naming it reads with the new value on the next pass, with nothing to update per image.
- Repointing an image at a secret the repository will not accept is refused at the dialog, the same as at registration.
- Deleting a secret an image names is refused, and the error names the image. Deleting the image first and then the secret succeeds.
- Deleting an image leaves the secret it named alone.
- The Console never sends a registry password to the Images service, and no response from it carries one.

## Notes

- **The organization check is the point, not a formality.** `ResolveSecretExists` returns the secret's organization, and the Images service compares it. Existence alone would let an owner name a secret they cannot read, point the image at a registry they control, and collect the value from the next discovery pass. The [LLM service's](../architecture/llm.md) equivalent check does not compare organizations today; it should, and that is a separate change.
- **`password` stays in the proto, deprecated.** Removing the field would be a breaking change to a published module for no gain; the service ignores it, and every caller in the platform sends `secret_id`.
- **A platform image registers with no credential.** `RegisterPlatformImage` runs at install, before anything has created a Secret, and what a release ships is readable anonymously. The field is gone from the provisioning config rather than left as something that cannot work.
- **The Terraform provider gains drift detection it could not have.** `password` was write-only and never round-tripped, so state kept whatever was last configured. `secret_id` is returned on read, so a credential changed outside Terraform is now visible in a plan.
- **The Secrets service performs no authorization checks of its own.** Noticed while wiring the reference check: `CreateSecret`, `GetSecret`, and the rest have no permission check in the handler, though [Secrets — Authorization](../architecture/secrets.md#authorization) specifies one for each. Out of scope here and worth its own change.
