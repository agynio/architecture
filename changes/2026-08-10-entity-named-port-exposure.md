# Entity-Named Port Exposure

## Target

- [Expose Service — Hostname](../architecture/expose-service.md#hostname)
- [Expose Service — Exposure Resource](../architecture/expose-service.md#exposure-resource)
- [Expose Service — Authorization](../architecture/expose-service.md#authorization)
- [Port Exposure — Link Format](../product/port-exposure/port-exposure.md#link-format)
- [Sandboxes — What's Inside](../product/sandboxes/sandboxes.md#whats-inside)
- [Identity — Interface](../architecture/identity.md#interface)
- [Organizations — Slug](../architecture/organizations.md#slug)
- [Resource Definitions — Sandbox](../architecture/resource-definitions.md#sandbox)
- [OpenZiti — API Surface](../architecture/openziti.md#api-surface)
- [Authorization — Expose Service](../architecture/authz.md#expose-service)
- [`agyn` CLI — Port Exposure Commands](../architecture/agyn-cli.md#port-exposure-commands)

## Delta

**An exposed port is addressed by a UUID that names nothing.** `http://exposed-7f3a2c91-….agyn:3000` says only that something, somewhere, is serving on port 3000. A link pasted into a conversation carries no indication of which agent produced it; a link to a sandbox dev server cannot be bookmarked, because the identifier is per-exposure and a workload restart mints a new one. The address was designed before agent instances and sandboxes existed, when the only thing behind an exposure was an anonymous per-thread agent workload.

Exposures are now addressed by the entity whose workload serves them:

| Owner kind | Hostname | Example |
|---|---|---|
| `sandbox` | `<sandbox.name>.<org.slug>.agyn` | `super-sandbox.acme.agyn` |
| `agent_instance` | `<instance_suffix>.<nickname>.<org.slug>.agyn` | `research.bob.acme.agyn` |
| either, when no readable form is derivable | `exposed-<exposure.id>.agyn` | unchanged from today |

The instance form is the `@bob#research` [handle](../architecture/agent-instances.md#handles) read back to front. Structuring it as `<suffix>.<nickname>` rather than flattening it to `bob-research` is what keeps the two namespaces from colliding: a sandbox occupies one leaf under the organization label, an agent class occupies everything beneath its own label, so a sandbox named `bob` and an agent nicknamed `@bob` coexist with no coordination between the services that own those names.

**The organization slug is always present**, which also puts every readable address at three labels or more — it can never collide with a platform service name (`gateway.agyn`, `llm-proxy.agyn`, `tracing.agyn`), a property the two-label form `super-sandbox.agyn` would not have as the platform adds services.

**No random component is added.** Uniqueness comes entirely from constraints that already exist — cluster-unique org slugs, org-unique sandbox names, `UNIQUE(org_id, nickname, instance_suffix)` on the nickname index — so a hash would buy nothing but noise in the one string this change exists to make readable. Its only residual use, disambiguating a name reused after a sandbox is deleted, is a same-organization window that the spec names and accepts.

### Supporting changes

**The Exposure record is not agent-shaped.** It carries `agent_id` and authorizes on `workload.agent_identity_id`, while [Sandboxes](../product/sandboxes/sandboxes.md#whats-inside) already promises `agyn expose add` at a sandbox shell — a promise nothing in the Expose service can currently keep. `agent_id` becomes `owner_kind` + `owner_id`, matching the generalization [Runners](../architecture/runners.md#workload-resource) already made, plus `organization_id` and the resolved `hostname`. The self-service authorization check becomes identity equality against the workload, which reads identically for both owner kinds and needs no relation on either.

**`Identity.BatchGetNicknames` is recorded as the reverse-lookup path.** It is the only route to an instance's full handle, since a system-generated `instance_suffix` lives on the nickname index rather than on the instance record. The RPC already exists and is unchanged; what was missing is the spec naming it, and any consumer using it to build an address.

**`UpdateService` gains the Expose service as a caller.** The address is derived from mutable inputs, so reconciliation re-derives every live exposure's hostname each pass and rewrites the `intercept.v1` config on drift. This is what carries an organization, sandbox, or instance rename through to live exposures — at the cost of breaking links already shared under the old name, which the specs state at every surface where a person can act on it.

**Two name fields are tightened to valid DNS labels.** The organization slug allows 64 characters where a DNS label permits 63, and both it and the sandbox name allow leading and trailing hyphens, which a hostname does not. Nicknames and instance labels are *not* tightened — they permit `_`, which DNS does not, and they are `@mention` handles in active use; an exposure whose nickname or suffix contains `_` takes the opaque fallback instead. Degrading is deliberate: a missing or underscore-bearing nickname is a cosmetic gap, and refusing to expose a port over one would turn it into an outage.

**`AddExposure` becomes idempotent per `(workload_id, port)`.** Exposures on one entity share a hostname and differ only in port, so a second record for an already-exposed port would be a duplicate OpenZiti intercept rather than a second address — a conflict at the Controller, not a harmless duplicate as it was under per-exposure addresses.

## Acceptance Signal

- `agyn expose add 3000` in a sandbox named `super-sandbox` in organization `acme` returns `http://super-sandbox.acme.agyn:3000`, and an enrolled device opens it.
- The same command in a workload for `@bob#research` returns `http://research.bob.acme.agyn:3000`; for an unlabelled instance of `@bob` it returns the instance's generated suffix under `bob`.
- An organization with a sandbox named `bob` and an agent nicknamed `@bob`, both exposing port 3000, produces two working addresses and no Controller conflict.
- Stopping a sandbox, restarting it, and re-exposing port 3000 returns the identical URL, and a bookmark taken before the stop resolves after it.
- `agyn expose add 3000` run twice returns the same URL both times and leaves one exposure in `agyn expose list`.
- Exposing a port from an instance whose class has no nickname, or whose nickname contains `_`, returns an `exposed-<id>.agyn` URL that works.
- Renaming the organization causes live exposure addresses to change within one reconciliation interval; the new address resolves and the old one stops resolving.
- `CreateOrganization` rejects a 64-character slug and a slug with a trailing hyphen; `CreateSandbox` rejects a name with a leading hyphen.
- An exposure record shows `owner_kind`, `owner_id`, and the resolved `hostname`, and the OpenZiti service behind it is still named `exposed-<id>` and still tagged to the Expose service.
- A cluster admin can still expose a port on behalf of a running workload by passing `workload_id` explicitly; a non-admin passing it is refused.

## Notes

- **Dial policies are unchanged and still `#all`.** This change makes addresses guessable and therefore removes the accidental protection an unguessable UUID was providing; it does not narrow who may reach an exposed port. That is [its own open question](../open-questions.md#port-exposure-scoped-access-control), now carrying the two sub-questions this change surfaces — whether sandboxes and instances want the same rule, and whether org-level scoping is a sufficient first step. Deliberately not resolved here: a naming change should not smuggle in a reconciled per-participant policy subsystem.
- **The OpenZiti service name stays `exposed-<id>`.** Only the `intercept.v1` address is readable. Ownership already resolves through [resource tags](../architecture/openziti.md#openziti-resource-tagging) rather than names, so a rename touches one config object and no policy, role attribute, or object identity.
- **Rename propagation is polled, not evented.** Reconciliation already re-reads every live exposure and the derivation inputs are three cheap lookups, so a subscription to Organizations, Agents, and Identity buys only a shorter stale window on a cosmetic value.
- **The reuse window is accepted, not solved.** A terminated sandbox releases its name; a new sandbox claiming it inherits the address, and a link held from before reaches the new sandbox. Both are in the same organization. Whether `name` uniqueness should span terminated sandboxes is a Sandbox lifecycle question and is not decided here.
- **Nothing about the OpenZiti object model changes.** One service, one Bind policy, one Dial policy per exposure; the same lifecycle, the same reconciliation triggers, the same cleanup ordering. The delta is the string in `intercept.v1` and the fields that produce it.
- **The nickname reverse lookup was undocumented, not missing.** `BatchGetNicknames` ships in `agynio/api` today while [Identity — Interface](../architecture/identity.md#interface) listed only the forward direction. Recorded here because Expose depends on it; other consumers that need to render a system-generated handle from an ID were already able to.
