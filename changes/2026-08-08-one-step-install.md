# One-Step Install

## Target

- [Platform Installation](../architecture/operations/platform-installation.md)
- [Platform Resource Provisioning](../architecture/operations/platform-provisioning.md)
- [Authorization — Bootstrap](../architecture/authz.md#bootstrap)
- [Authentication — Identity Types](../architecture/authn.md#identity-types)
- [OpenZiti — Static Policies](../architecture/openziti.md#static-policies)

Supersedes [Platform Resource Self-Registration](2026-08-03-platform-self-registration.md), which specified a dedicated internal provisioning surface. The organization and image halves of it were built; this change replaces them rather than extending them to apps and runners.

## Delta

Installing the platform is two Helm releases with an operator step between them, and what provisioning exists is a script rather than a declaration. Everything below follows from removing both.

### Provisioning Model

Provisioning today is procedural: a Job runs a fixed sequence of calls once per install and once per upgrade, and everything the platform needs must be true by the time it runs.

- There is no declared desired state — nothing describes what the platform should contain, only code describing what to create. Nothing can be inspected, diffed, or corrected between runs.
- Convergence is a retry loop inside a single run, sized for a warm cluster: a handful of attempts over about a minute, after which the run reports failure and exits successfully so as not to fail the release. A cold install can outrun it, and the recovery is an operator running an upgrade.
- A run that gave up is indistinguishable from one that succeeded without reading its logs, so a platform that installed but never provisioned reports as healthy.
- Resources are attempted once per run rather than reconciled individually, so one dependency that is slow to appear costs every resource behind it.
- Drift is never corrected, and a release cannot change a resource it shipped earlier — provisioning is create-if-absent, so an image's metadata is whatever the first release that created it said.

Nothing in the platform is managed this way today: there are no custom resources and no controller anywhere in it.

### Provisioning Path

Provisioning runs through dedicated internal methods — `RegisterPlatformOrganization` and `RegisterPlatformImage` — called with no identity and restricted by Istio principal. The desired path is the ordinary API, called as a cluster admin.

- Both methods, and the Istio policies that guard them, are to be removed. Nothing else calls them.
- The provisioner calls services directly and holds no credential. It should call the [Gateway](../architecture/gateway.md), so provisioning is subject to the same authentication and authorization as any other caller.
- Create-if-absent is implemented inside each of those two methods. No service should carry a second creation path that exists only for install.

### Platform Admin Identity

`CLUSTER_ADMIN_TOKEN` and `CLUSTER_ADMIN_IDENTITY_ID` are accepted by the Gateway, but nothing behind them is real.

- No identity is registered in [Identity](../architecture/identity.md) for the configured ID.
- The `admin` on `cluster:global` tuple is written straight to OpenFGA by install glue rather than through the service that owns it.
- The Gateway resolves the token to an identity typed `user`. It has no record in Users, so it is a member of the system organization whose profile cannot be resolved — and nothing but the resolver's current shape stops a future change from provisioning it as a real account and spending the one-shot first-admin claim before any operator signs in. It needs an identity type of its own.
- The token is rendered from a literal Helm value into the Gateway workload spec, so it is readable by anyone who can read a Deployment and is replaced by the placeholder on every upgrade — [`agyn local upgrade`](../architecture/agyn-cli.md) carries a step that puts it back afterwards. It should be generated per install and held in a Secret, with an operator-managed mode like the other generated values have.

### Cluster Administrators

The role is claimed by the first account [Users](../architecture/users.md) provisions, narrowed by an optional configured address. It should be declared instead.

- Nothing declares who administers a cluster. With no address configured, whoever signs in first takes the role; with one configured, the claim silently never completes if the identity provider does not mark the address verified.
- The claim is one-shot and spent by whoever takes it, so losing the account is a recovery procedure rather than an edit. There is no declarative way to name a second administrator.
- The claim record and the authorization tuple live in different stores and cannot be written atomically. The chosen order fails toward a cluster with no admin — recoverable, but only by hand.
- Granting cluster admin needs no new method: `CreateUser` and `UpdateUser` already carry a cluster role, and `SearchUsers` already finds an account by address.

### Apps and Runner

Neither is provisioned. Both are registered by hand between the two releases, and the service token each needs is copied from the Console into a Kubernetes Secret by an operator.

- Nothing registers the runner or the bundled apps as part of installing the release.
- Nothing writes the service token Secrets those workloads mount; the provisioner has no Kubernetes write access at all today.
- A lost Secret is silent. Nothing reports that a registered runner or app has no readable credential.

### Overlay Authorization

The [static OpenZiti policies](../architecture/openziti.md#static-policies) are applied by hand or by install-specific glue before the platform can reach its own services. They are not part of any release, so an install that omits them produces services that are invisible rather than refused.

### Packaging

The platform ships as `agyn-platform` and `agyn-apps`, and the parameter interface of both exposes wiring rather than intent.

- Per-service environment lists are the override surface, and they replace rather than merge — an operator setting one value must re-declare every entry the chart supplied, including ones they never wanted to change.
- Roughly fifteen dependency paths must be kept consistent with the top-level values by hand, with render-time checks that exist to detect the drift. The consistency should not be expressible.
- Service addresses and secret key references are operator inputs in both charts, rather than being derived from the values that determine them.
- Bundled dependencies both ship and default to on, so a cluster already running one receives a second copy.

### Provisioner Home

The provisioner ships in the Images service repository and chart. Its scope is platform-wide, so it belongs at the platform level.

## Acceptance Signal

- The platform installs with one release and no operator step inside it.
- What the platform should contain is declared, inspectable, and diffable — not implied by code that has run.
- A freshly installed platform has the system organization, the platform's images, the runner and the bundled apps present, with nothing registered by hand.
- Installing onto a cold cluster provisions everything without an operator re-running anything, and an install whose provisioning has not completed says so per resource rather than reporting healthy.
- Changing a declared resource in a new release changes the resource; correcting one by hand against the API is reverted.
- Removing a declared organization, image, app or runner leaves it in place; removing a declared cluster administrator revokes the role.
- Cluster administrators are exactly those the install names. An install that names none has none, and naming one afterwards grants it.
- The overlay's static policies are present after install, with nothing applied by hand.
- No `RegisterPlatform*` method exists on any service.
- The platform admin identity is registered in Identity with a type of its own, holds its cluster tuple written by the service that owns it, and has no user account.
- The bootstrap token cannot be read from a workload spec, and survives an upgrade without a step that restores it.
- No provisioning path writes to a service database or to OpenFGA directly.
- A values file sets no service address, no secret key reference and no per-service environment list, and no render-time check validates consistency between two operator-set values.
- Enabling a bundled dependency is an explicit choice; a cluster already running one does not receive a second.

## Notes

- This introduces the first custom resources and the first controller in the platform. The cost is a new component class — CRD lifecycle, RBAC, status conventions — against removing install ordering, readiness gating, run deadlines and single-flight concerns entirely.
- Packaging the definitions and the declarations in one chart has two constraints worth settling before the layout is fixed: definitions must upgrade in place with the release, and must never be deleted and recreated, since that destroys every declaration of the kind. Whether a first install can apply a declaration in the same release that registers its kind is the thing to confirm early — if it cannot, the fallback puts an out-of-band schema step back into the upgrade path.
- Declarative management of these resources is not a new idea in the platform: [`terraform-provider-agyn`](../../terraform-provider-agyn) already models runners and their siblings. This is that idea in-cluster, with the platform as its first user rather than a special install path.
- The workload layer becomes version-locked to the platform release. That is intended for components the platform ships; third-party runners and apps are registered through the API and deployed independently.
- On a first install the runner and bundled apps may start before the controller has written the Secrets they mount, and retry until it does. Accepted: it needs no operator action.
- Losing a service token Secret leaves the runner or app registered with an unreadable credential, recoverable only by deleting and re-declaring it. Accepted: no service token in the platform has a rotation path, and this changes neither way.
- A fresh install still has no LLM provider or model. That is organization configuration rather than something a release ships.
- The bootstrap token is also what `agyn local` signs in with. Moving it into a Secret must keep that working; removing it entirely is blocked on the local bundle having another way to authenticate.
