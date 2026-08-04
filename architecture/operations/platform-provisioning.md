# Platform Resource Provisioning

## Overview

Some platform resources must exist for a freshly installed platform to be usable at all — the [images](../../product/images/images.md) the platform itself ships, chief among them. They cannot be created the ordinary way: the ordinary way requires a signed-in user with organization ownership, and at install time there is no user, no organization, and nothing for an operator to click.

Today this gap is filled by writing directly into service databases — the initial cluster admin is seeded by Terraform writing to PostgreSQL, per [Authorization — Bootstrap](../authz.md#bootstrap). That works exactly once and only for one resource, and it bypasses every validation the owning service performs.

This document specifies the general shape: an **internal provisioning API** that platform components use to declare the resources they need, authenticated as a service identity rather than a user.

## Principles

**Not exposed through the Gateway.** The provisioning surface is reachable only from inside the platform. It is not an alternative path to the public API and carries no user identity.

**Authenticated as a service identity.** The caller is the platform's own provisioning job, holding a platform identity like any other non-human actor. Organization membership is not required — provisioning predates it.

**Create-if-absent, never update.** A provisioning call creates a resource when nothing of that name exists and returns the existing one otherwise. It never overwrites. This is what makes re-running an upgrade safe: an operator who edited a provisioned resource keeps their edit, and the platform does not fight them for it on every release.

The consequence is accepted deliberately: a platform release cannot correct the metadata of a resource it provisioned previously. For [images](../images-service.md) this costs nothing, since a new release publishes new tags rather than editing old records.

**No ownership marker.** Provisioned resources are ordinary resources in an ordinary organization. There is no platform-owned flag, no read-only enforcement, and no special-casing anywhere in the API. An operator may edit or delete them; the next upgrade recreates what is missing and leaves what is not. The platform self-heals on upgrade, not continuously.

## The Platform Organization

Platform-provisioned resources live in an ordinary [organization](../organizations.md), created by the same provisioning path.

It has **no members**. Cluster admins already hold owner-level access to every organization on the platform (see [Authorization — Cluster Permissions](../authz.md#cluster-permissions)), so it is administrable without anyone being added to it, and no human identity needs to be manufactured at install time.

Its images are registered `public`, which is what makes them usable from every organization on the platform.

## Scope

The same mechanism covers every platform component that must register itself:

| Component | Provisioned | Alternative for non-platform instances |
|---|---|---|
| **Images** | The platform's workspace and agent runtime image records | Organization owners register their own through the Console |
| **Runners** | The runner shipped with the platform | Operators enroll external runners with a token from the Console |
| **Apps** | The apps shipped with the platform | Developers publish their own through the Apps service |

In each case the provisioning path and the user-facing path create the same resource, and neither is privileged over the other afterwards.

## Execution

Provisioning runs as part of installing or upgrading the platform, after the services it calls are available. It is idempotent by construction (create-if-absent), so it runs on every install and every upgrade without a guard.

A component whose provisioning fails does not block the platform from starting; the resource is simply absent, and the next upgrade attempts it again.

## Related Architecture

- [Images Service — Internal Registration](../images-service.md#internal-registration)
- [Images (product)](../../product/images/images.md)
- [Organizations](../organizations.md)
- [Local Bundle](local-bundle.md)
- [Authorization — Bootstrap](../authz.md#bootstrap)
