# Environment as a Private Resource Access Principal

## Target

- [Private Networks — Why an environment is a principal](../architecture/private-networks.md#why-an-environment-is-a-principal)
- [Private Networks — Dial Policy (per Access Grant)](../architecture/private-networks.md#dial-policy-per-access-grant)
- [Private Networks — Role Attributes](../architecture/private-networks.md#role-attributes)
- [Networks Service — PrivateResourceAccess CRUD](../architecture/networks-service.md#privateresourceaccess-crud)
- [Networks Service — Events Consumed](../architecture/networks-service.md#events-consumed)
- [Resource Definitions — Private Resource Access](../architecture/resource-definitions.md#private-resource-access)
- [Authorization — Networks Service](../architecture/authz.md#networks-service)
- [Private Networks — Granting access](../product/private-networks/private-networks.md#granting-access)

## Delta

**A sandbox cannot be granted access to a private resource, and no principal type can express one.** `PrivateResourceAccess` accepts `agent`, `user`, `app`, and `group`. A sandbox workload matches none of them: it carries no `agent-<id>` attribute because there is no agent class behind it, and it cannot be a group member because groups collect users, agents, and apps. The engineer at a sandbox shell — the caller most likely to want an internal Postgres — is the one principal the access model has no way to name.

`principal_type` gains `environment`. A grant to an environment resolves to every workload running it, agent workloads and sandboxes alike.

**Nothing new is provisioned in OpenZiti.** The [Agents Orchestrator](../architecture/agents-orchestrator.md#workload-spec-assembly) already stamps `environment-<environmentId>` on every workload identity it creates, which is the attribute [egress rule attachments](../architecture/egress-rules-service.md) target for exactly this reason. The grant's Dial policy names `#environment-<id>` and the rest of the topology is unchanged — same per-resource service, same per-grant policy, same reconciliation, same ≤15s propagation. A workload already running picks the grant up on its next service-list poll; nothing restarts, because the attribute predates the policy.

### Supporting changes

**An environment is not an identity**, so the two per-principal steps that assume one need a branch. Existence and type validation go to `Agents.GetEnvironment` rather than `Identity.GetIdentityType` — the Networks service gains the [Agents](../architecture/agents-service.md) service as a dependency and `AGENTS_SERVICE_ADDRESS` as configuration. The cross-org guard is unchanged in shape: the [`environment` OpenFGA type](../architecture/authz.md#environment) carries `org` like every other principal, so the same `Check` covers it.

**Authorization is `can_edit_config` on the environment**, matching what an [egress rule attachment](../architecture/egress-rules-service.md#authorization) to an environment already requires, and matching the `agent` principal's use of `can_edit_config` on the agent. Not `owner` on the organization: an environment grant is the same class of act as attaching a credential-injecting egress rule to that environment, and the platform already settled that at `can_edit_config`.

The consequence is stated at every surface where someone can act on it: a grant to an environment reaches anyone holding `can_use` on it, because they can start a sandbox and a sandbox is a shell. It adds a destination to the set a shell there already reaches — the environment's secret-backed ENVs, its egress rules, its volumes — rather than widening who is standing in front of that set.

## Acceptance Signal

- An operator grants a private resource to an environment; a sandbox started in that environment dials the resource by its intercept hostname and connects.
- An agent whose environment holds the grant dials the same resource, with no grant of its own.
- A workload already running when the grant is created picks it up within ≤15s, without restarting.
- Revoking the grant stops both within ≤15s and tears down in-flight connections.
- `CreatePrivateResourceAccess` with an `environment` principal is refused for a caller holding `can_use` but not `can_edit_config` on that environment.
- The same call is refused when the environment belongs to a different organization than the resource.
- A grant naming an environment that does not exist is refused at create time.
- `ListPrivateResourceAccess` filtered by `principal_type=environment, principal_id=<id>` returns the grant.
- Deleting the environment leaves the grant behind and grants nothing: no workload can carry `environment-<deleted-id>`.

## Notes

- **Agent instances and individual sandboxes are deliberately out of scope.** Both are narrower than an environment and neither has a role attribute that outlives a workload — `workload-<id>` dies with the pod, so a grant against it would not survive a restart. Naming either as a principal needs a durable per-instance or per-sandbox attribute first, which is its own design.
- **No cleanup event exists for a deleted environment.** The `group` principal has `agyn.groups.group.deleted`; [Agents](../architecture/agents-service.md) publishes no environment lifecycle event to mirror it, so a grant to a deleted environment is left in place. It is inert rather than dangerous — `environment-<id>` is stamped only on workloads running that environment, and [deletion is blocked](../architecture/resource-definitions.md#environment) while any agent or sandbox references it, so no identity can ever carry the attribute the orphaned policy names. Closing the hygiene gap wants an `agyn.agents.environment.deleted` event, not a per-pass existence check on every grant.
- **Environments do not become group members.** A group collects identities; an environment is a configuration resource. Granting a group and granting an environment stay separate mechanisms.
- **No Console work is specified here.** The access-list UI on the resource detail page needs an environment picker alongside the existing principal pickers; that is a Console change and is left to the [Console spec](../product/console/console.md).
