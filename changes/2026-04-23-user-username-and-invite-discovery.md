# User Username and Invite Discovery

## Target

- [Users — Username](../architecture/users.md#username)
- [Users — User Directory](../architecture/users.md#user-directory)
- [Organizations — Default Nickname on Activation](../architecture/organizations.md#default-nickname-on-activation)
- [Console — Members](../product/console/console.md#members)
- [Identity — Nickname Index](../architecture/identity.md#nickname-index)
- [Terraform Provider — User Resource](../architecture/operations/terraform-provider.md#user-resource)

## Delta

The Console invite form promises that owners can "search existing platform users by name or email" ([console.md](../product/console/console.md)), but no public endpoint backs that search. `ListUsers`, `GetUser`, and `GetUserByOIDCSubject` are cluster-admin only; `ResolveNickname` is org-scoped and internal. Org owners therefore have no way to discover an `identity_id` for someone who is not already in their org, blocking the documented invite flow.

This change introduces a cluster-wide `username` on the User model and a public `SearchUsers` endpoint to back invite discovery:

- **User model** — adds `username`: cluster-wide unique, pattern `^[a-z0-9_-]+$`, max 32 chars, freely renameable. Distinct from per-org `nickname`.
- **Provisioning** — `ProvisionUser` derives `username` deterministically from OIDC claims (`preferred_username` → `email` local part → `name` → hash of `oidc_subject`), normalizes, and resolves collisions with numeric and random suffixes.
- **Public search endpoint** — `SearchUsers(prefix, limit)` on the Users service. Authenticated callers only, prefix match on `username`, returns redacted profiles (no `oidc_subject`, no email, no `cluster_role`), limit ≤ 20, rate-limited.
- **Default org nickname on activation** — when a user's membership becomes active (via `AcceptMembership` or direct creation), Organizations seeds an `org_nicknames` row from the user's current `username` (best-effort; skipped on conflict).
- **Console** — Members invite form now searches by `username` rather than name/email; Profile menu shows `username` as editable.
- **Terraform** — `agyn_user` resource gains an optional `username` input (computed when omitted); `data.agyn_user` exposes `username` as a computed output. `nickname` is removed from the `agyn_user` schema (it was incorrect — nicknames are per-org and not part of the user record).

Out of scope: inviting users who do not yet exist on the platform. The invitee must sign in once before they can be discovered.

## Acceptance Signal

- New users provisioned via OIDC receive a non-empty, unique `username` derived from their claims with no manual step.
- An authenticated, non-admin user can call `SearchUsers("ali", 10)` through the Gateway and receive matching users with redacted profiles.
- Console Members → Invite member resolves a username via `SearchUsers`, the owner picks a result, and `CreateMembership` is called with the resulting `identity_id`.
- After `AcceptMembership` or a cluster-admin direct add, the new member has an `org_nicknames` row equal to their current `username` (or no row if the username collides in that org).
- Renaming a user's `username` does not modify any existing `org_nicknames` rows.
- Cluster admins creating users via Terraform either supply `username` explicitly or get a derived one back as a computed value.
