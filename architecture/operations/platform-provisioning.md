# Platform Resource Provisioning

## Overview

Some resources must exist for a freshly installed platform to be usable at all: an organization for the platform's own resources to live in, the [images](../../product/images/images.md) it ships, the [runner](../runners.md) that executes agent workloads, the [apps](../apps.md) bundled with the release, and at least one cluster administrator. None of them can be created the ordinary way at install time — the ordinary way requires a signed-in user with organization ownership, and at install time there is no user and no organization.

The platform closes that gap **declaratively**. A release declares the resources it needs as Kubernetes objects, and a controller reconciles them against the platform's own API, authenticating as a [platform admin identity](#the-platform-admin-identity). There is no install script and no ordered sequence of calls: an object that cannot be reconciled yet is retried until it can be.

This is what makes the platform installable in one step. Everything the release ships is declared by the release; the ordinary APIs remain what third parties use to bring their own runners, apps and images.

## Principles

**Declared state, not executed steps.** What a release provisions is a set of objects, not a procedure. Nothing depends on the order the objects are applied in, on anything having finished first, or on a run completing — the same declaration converges from any starting point, including a half-provisioned one.

**Ordinary resources through the ordinary API.** There is no separate provisioning surface. The controller calls the same [Gateway](../gateway.md) methods the Console calls, with the same authentication and the same authorization checks. A provisioned resource is indistinguishable from one an operator created, because it was created the same way.

**The declaration is authoritative.** The controller reconciles content, not just existence: an image's tag filter or an app's visibility is whatever the object says, and a change made directly against the platform API is corrected on the next pass. Edits belong in the declaration. This is what lets a release change a resource it shipped earlier, which a create-if-absent scheme cannot do.

**Removal revokes grants and orphans resources.** Removing a declaration is not a request to destroy data. An organization, image, app or runner that leaves the declared set is left in place, because deleting an app would take everything it owns with it. A [cluster admin](#cluster-administrators) grant is the exception and is revoked, because an unrevokable grant is a hole rather than a resource.

## The Platform Admin Identity

One identity, registered in [Identity](../identity.md) and holding `admin` on `cluster:global` (see [Authorization — Cluster Permissions](../authz.md#cluster-permissions)). It is what the controller authenticates as, the identity every provisioned resource is created by, and therefore the owner of the [system organization](#the-system-organization).

Its credential is a **bootstrap token** configured on the Gateway, which resolves it directly rather than through [Users](../users.md):

| | |
|---|---|
| Configuration | `CLUSTER_ADMIN_TOKEN` and `CLUSTER_ADMIN_IDENTITY_ID` on the Gateway |
| Resolution | A fourth path alongside OIDC, [API token](../api-tokens.md) and OpenZiti — a constant-time comparison against the configured value |
| Resolves to | The configured identity, with an identity type of its own — not `user` |

It is generated per install and stored as a Kubernetes Secret rather than rendered into a workload spec, so it is not readable by anyone who can read a Deployment and survives an upgrade re-rendering the release. Rotation is replacing the Secret and restarting the Gateway — the token is compared, never stored hashed, so there is no second copy to reconcile.

**It is a standing cluster-admin credential on the public API surface.** Anything holding it can do anything a cluster admin can. This is the deliberate cost of provisioning through the ordinary API rather than through a privileged internal one: the exposure is one credential, in one Secret, with a rotation path — rather than a set of internal write methods that bypass authorization on every service that owns a provisioned resource.

**It is not a user, and must never become one.** It has no account in [Users](../users.md) and no profile, and it is typed accordingly. An identity typed `user` with no user record is a member the Console cannot name — and, more seriously, anything that provisioned it as a real account would be creating the platform's first user account, at install time, before any human has signed in.

## Cluster Administrators

Cluster admins are **declared**, not claimed. Each declaration names a person by the address their identity provider will assert, and the controller grants `admin` on `cluster:global` to the matching account.

A declaration is satisfied when that account exists. Before the person's first sign-in there is no account to grant against, so the declaration is *pending* and the controller keeps retrying — signing in is what completes it, not what triggers it. Nothing else grants the role, and an unnamed account never receives it regardless of when it appears.

This replaces a first-sign-in claim, and is better in every dimension that matters:

| | First-sign-in claim | Declaration |
|---|---|---|
| Who becomes admin | Whoever signs in first, unless narrowed by configuration | Exactly who is named |
| More than one | A second admin is granted by the first, by hand | Declare more |
| Losing the account | A recovery procedure — the claim is spent and does not reopen | Declare another |
| Partial failure | The claim record and the grant live in different stores and cannot be written atomically; the orders fail differently and one of them can leave a cluster with no admin | Retried until both agree |

The consequence is accepted deliberately: **an install that declares no administrator has none.** Nobody is granted the role by arriving first. That is a deterministic and recoverable state — apply a declaration — rather than a race whose outcome depends on who opened the Console.

## The System Organization

Platform-provisioned resources live in an ordinary [organization](../organizations.md), created through `CreateOrganization` like any other.

The creator becomes its owner, so the platform admin identity holds `owner` on it. That ownership is what lets the controller go on to create apps and images in it without any service granting an exception. A cluster admin may also administer it without being a member (see [Authorization — Cluster Permissions](../authz.md#cluster-permissions)).

Its images are registered `public`, which is what makes them usable from every organization on the platform.

Its slug is cluster-wide unique and user-visible — it appears in [image references](../image-proxy.md#reference-rewriting) and [app addresses](../apps.md#identification) — so it is a deliberate choice rather than an implementation detail.

## Scope

| Declared | Reconciled against | Alternative for non-platform instances |
|---|---|---|
| **System organization** | `CreateOrganization` | Users create their own through the Console |
| **Cluster admins** | The [Users](../users.md) cluster role | An existing admin grants the role through the Console |
| **Images** | `CreateImage`, `public` | Organization owners register their own through the Console |
| **Runner** | `RegisterRunner`, cluster-scoped | Operators enroll external runners with a token from the Console |
| **Apps** | `CreateApp` | Developers publish their own through the [Apps service](../apps-service.md) |
| **Overlay policies** | The [OpenZiti](../openziti.md#static-policies) controller | — the platform's own baseline connectivity |

Overlay policies are the one kind not reconciled against the platform API: they are configuration on the OpenZiti controller rather than a platform resource. They are declared alongside the rest because their failure mode is identical — [OpenZiti](../openziti.md) denies by default, so a missing policy makes a service invisible rather than refused — and because a policy naming a service cannot be applied until that service exists, which is exactly the kind of "not yet" that reconciliation absorbs.

## Service Tokens

Registering a runner or an app mints a **service token**, returned once and stored hashed. The controller writes it into the Kubernetes Secret the workload mounts and records the reference on the object's status, which is what lets the runner and the bundled apps [enroll](../runners.md#enrollment) on first start without an operator copying a credential between two installs.

Because the token is returned once, its Secret is the only copy. The controller does not re-register a runner or app whose record already exists in order to obtain a new one.

**A lost Secret is not recoverable in place.** The record exists, its token is stored only as a hash, and no method reissues one. The object reports the credential as missing rather than silently converging, and recovery is deleting the runner or app so the declaration recreates it. This is the same gap third-party runners and apps have today — no service token in the platform has a rotation path — and provisioning neither improves nor worsens it.

## Reconciliation

The controller runs against a platform that may still be starting, may have drifted, or may have been partly provisioned by an earlier release. All three are the same situation: compare the declaration to what exists, and act on the difference.

**A failed call is a retry, not an error.** The controller does not probe for readiness or wait for a signal. On a fresh install almost every failure means "not yet" — a service still starting, a schema still migrating, an overlay identity not yet enrolled — so the object is requeued and tried again.

**Objects are independent.** Each reconciles on its own, so the organization and the platform's images are not held back by an [OpenZiti](../openziti.md) controller that only the runner and apps need. The only true dependency is the organization, because the rest are created inside it; an object that needs it waits on its status rather than on a step having run.

**Progress is per-object and durable.** What has been created is recorded on each object's status, so nothing is inferred from whether a previous run finished. There is no partial state to clean up and no ordering to re-establish.

**State is reported, not logged.** Every object carries conditions saying whether it is reconciled, pending, or failing and why. A platform that installed without provisioning is visible as objects that are not ready — not as a failed job whose logs have to be read, and not as a successful install hiding an unregistered runner.

Because nothing about this depends on an install or upgrade boundary, the platform corrects drift continuously rather than only when a release moves.

## Out of Scope

Two things an install needs that this mechanism does not cover:

- **Cluster prerequisites.** Kubernetes, the service mesh, DNS, TLS and the identity provider are what a release is installed onto. See [Platform Installation](platform-installation.md#outside-the-chart).
- **LLM providers and models.** An organization's own configuration, not something a release ships. A freshly installed platform has none until someone adds one.

## Related Architecture

- [Platform Installation](platform-installation.md)
- [Authorization — Cluster Permissions](../authz.md#cluster-permissions)
- [Authentication](../authn.md)
- [Users](../users.md)
- [Organizations](../organizations.md)
- [Images Service](../images-service.md)
- [Apps Service](../apps-service.md)
- [Runners](../runners.md)
- [OpenZiti Integration — Static Policies](../openziti.md#static-policies)
- [Local Bundle](local-bundle.md)
