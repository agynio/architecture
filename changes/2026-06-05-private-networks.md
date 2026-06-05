# Private Networks

## Target

- [Product — Private Networks](../product/private-networks/private-networks.md)
- [Architecture — Private Networks](../architecture/private-networks.md)
- [Architecture — Networks Service](../architecture/networks-service.md)
- [Architecture — Groups Service](../architecture/groups-service.md)
- [Resource Definitions — Network](../architecture/resource-definitions.md#network)
- [Resource Definitions — Tunnel Credential](../architecture/resource-definitions.md#tunnel-credential)
- [Resource Definitions — Private Resource](../architecture/resource-definitions.md#private-resource)
- [Resource Definitions — Private Resource Access](../architecture/resource-definitions.md#private-resource-access)
- [Resource Definitions — Group](../architecture/resource-definitions.md#group)
- [Resource Definitions — Group Membership](../architecture/resource-definitions.md#group-membership)
- [OpenZiti — Tunnel Identity Lifecycle](../architecture/openziti.md#tunnel-identity-lifecycle)
- [OpenZiti — Group Role Attribute Sync](../architecture/openziti.md#group-role-attribute-sync)
- [Authorization — group type](../architecture/authz.md#group)
- [Authorization — Networks Service](../architecture/authz.md#networks-service)
- [Authorization — Groups Service](../architecture/authz.md#groups-service)
- [Users — Device OpenZiti Identity](../architecture/users.md#device-openziti-identity)
- [Identity — Related Concepts](../architecture/identity.md#related-concepts)

## Delta

There is no mechanism today for users to give agents access to resources inside their own private networks. Agents can reach the public internet (filtered by [EgressRule](../architecture/egress-rules-service.md) when configured) and platform-internal services over OpenZiti, but a host behind the operator's NAT — an internal Postgres, a private GitLab, a SaaS replica in a peered VPC — is unreachable. There is also no platform-level concept of identity groups: every permission and access grant must be written one identity at a time, which doesn't scale to enterprise organizations and blocks future SCIM provisioning.

The new Networks service, the Groups service, the user-managed OpenZiti tunneler model, and the extended OpenZiti role-attribute scheme collectively close this gap. The desired state is described in the documents linked above; this delta enumerates what's missing.

- **Networks service** — `agynio/networks` repository does not exist. The service owns `Network`, `TunnelCredential`, `PrivateResource`, and `PrivateResourceAccess` resources, their PostgreSQL storage, the per-network OpenZiti Bind policy lifecycle, the per-resource OpenZiti service + configs lifecycle, the per-grant OpenZiti Dial policy lifecycle, tunnel liveness polling against the OpenZiti Controller, the reconciliation loop, and change-event publishing. Does not exist in any form today.

- **Groups service** — `agynio/groups` repository does not exist. The service owns `Group` and `GroupMembership` resources, their PostgreSQL storage, OpenFGA tuple writes on the `group` type, OpenZiti role-attribute sync (`group-<id>` on App, User-device, and live Agent workload identities), reconciliation, and change-event publishing. Provides `ListMemberGroups` (Gateway-exposed, for the [Agents Orchestrator](../architecture/agents-orchestrator.md), [Networks service](../architecture/networks-service.md), and [Users service](../architecture/users.md)) and `ListMemberGroupsBatch` (internal-only). Does not exist today.

- **OpenZiti `group` type in the authorization model** — the OpenFGA model in `agynio/authorization` does not include the `group` type. Must be added with `org: [organization]`, `member: [identity]`, `admin: [identity]`, and the computed `can_view` / `can_edit` relations. Existing direct-role definitions on the `agent` type must be extended from `[identity]` to `[identity, group#member]` to allow group-based agent role grants. Future types that need group-based grants extend their role definitions similarly.

- **Ziti Management RPCs** — three new methods. `CreateTunnelIdentity(network_id)` creates a `Device`-type OpenZiti identity with `roleAttributes: ["tunnels", "network-<id>"]` and returns the identity ID + enrollment JWT. `DeleteTunnelIdentity(identity_id)` deletes a tunnel identity. `PatchIdentityRoleAttributes(identity_id, add, remove)` adds and/or removes role attributes on an existing identity without re-enrollment. None of these exist today.

- **OpenZiti role-attribute scheme extensions** — App identities gain `app-<appId>`, User device identities gain `user-<userId>` and per-group `group-<groupId>` attributes, Agent workload identities gain per-group `group-<groupId>` attributes at orchestrator-driven identity creation time, and a new Tunnel identity type carries `["tunnels", "network-<networkId>"]`. The role-attribute updates on existing identities are sourced via the new `PatchIdentityRoleAttributes` RPC.

- **`Network`, `TunnelCredential`, `PrivateResource`, `PrivateResourceAccess` resource shapes** — see [Resource Definitions](../architecture/resource-definitions.md). Key constraints: `Network.name` unique per org; `TunnelCredential.enrollment_jwt` returned once at creation; `PrivateResource.intercept_host` rejects reserved zones (`*.ziti`, `*.svc`, `*.cluster.local`, anything overlapping `100.64.0.0/10`); `(organization_id, intercept_host, intercept_port)` unique across resources; `intercept_ports` cardinality matches `target_ports` 1:1; `PrivateResourceAccess` unique on `(private_resource_id, principal_type, principal_id)`. Principals: `agent`, `user`, `group`. Apps not allowed in v1 (tracked in [Open Questions](../open-questions.md)).

- **`Group`, `GroupMembership` resource shapes** — see [Resource Definitions](../architecture/resource-definitions.md#group). Members are `user`, `agent`, or `app` (runners excluded). Flat membership only in v1. `Group.source` and `GroupMembership.source` discriminate `platform` vs `scim` ownership in preparation for IdP-driven sync; SCIM endpoints themselves are not part of v1.

- **Proto definitions in `agynio/api`** — new package `networks/v1` with `Network`, `TunnelCredential`, `PrivateResource`, `PrivateResourceAccess` messages and the `NetworksGateway` service (`CreateNetwork`, `GetNetwork`, `ListNetworks`, `UpdateNetwork`, `DeleteNetwork`, `CreateTunnelCredential`, `ListTunnelCredentials`, `DeleteTunnelCredential`, `CreatePrivateResource`, `GetPrivateResource`, `ListPrivateResources`, `UpdatePrivateResource`, `DeletePrivateResource`, `CreatePrivateResourceAccess`, `DeletePrivateResourceAccess`, `ListPrivateResourceAccess`). New package `groups/v1` with `Group`, `GroupMembership` messages and the `GroupsGateway` service (`CreateGroup`, `GetGroup`, `ListGroups`, `UpdateGroup`, `DeleteGroup`, `AddMember`, `RemoveMember`, `ListMembers`, `ListMemberGroups`). Internal-only `ListMemberGroupsBatch` not exposed through the Gateway.

- **Per-resource OpenZiti service provisioning** — the [Networks service](../architecture/networks-service.md) must call [Ziti Management](../architecture/openziti.md) `CreateService` on `CreatePrivateResource` (name `private-<id>`, role attribute `network-resources-<network_id>`, attached `intercept.v1` with `addresses: [intercept_host]` and `portRanges: intercept_ports`, attached `host.v1` with `address: target_host` and `portRanges: target_ports`) and `UpdateService` on config-affecting field changes, `DeleteService` on `DeletePrivateResource`. The per-network Bind policy (`identityRoles: ["#network-<id>"]`, `serviceRoles: ["@network-resources-<id>"]`) is created on `CreateNetwork` and deleted on `DeleteNetwork`. Per-grant Dial policy (`identityRoles: ["#<principal-role>"]`, `serviceRoles: ["@private-<resource_id>"]`) created on `CreatePrivateResourceAccess`, deleted on `DeletePrivateResourceAccess`. Reconciliation loop repairs drift.

- **Tunnel enrollment + liveness** — tunnel enrollment uses the standard OpenZiti tunneler distributions (`ziti-edge-tunnel`, the desktop apps, the Kubernetes helm chart). The platform issues the JWT and tracks the identity; the operator runs the tunneler. Tunnel liveness is sourced from the OpenZiti Controller's per-identity session API; the Networks service polls and emits `tunnel_status.changed` notifications on transitions. No platform-side heartbeat protocol with the tunneler.

- **OpenFGA tuple writes for groups** — group creation, membership add/remove, and group deletion drive OpenFGA writes per [Authorization — Tuple Lifecycle](../architecture/authz.md#tuple-lifecycle). Group deletion additionally requires every consuming service that wrote `group:<id>#member, *, *` grants to clean them up — implemented via subscription to `group.updated` deletion events.

- **OpenZiti role-attribute sync workers** — the [Groups service](../architecture/groups-service.md#openziti-role-attribute-sync) must patch role attributes on member identities on every membership change. The orchestrator must query `Groups.ListMemberGroupsBatch` when assembling per-workload agent identities. The Users service must query `Groups.ListMemberGroups` when enrolling new user devices and include the resulting `group-<id>` attributes. None of these call sites exist today.

- **Authorization model — Networks and Groups sections** — the [Authorization](../architecture/authz.md) doc gains "Networks Service" and "Groups Service" sections enumerating per-operation checks. The agent type adds `group#member` to its direct-role definitions. No other OpenFGA structural changes.

- **Notifications event types** — `network.updated`, `tunnel_credential.updated`, `tunnel_status.changed`, `private_resource.updated`, `private_resource_access.updated`, `group.updated`, `group_membership.updated`. Published by the Networks and Groups services to org-scoped rooms. Subscribers: Console (UI reactivity), Networks service (consume `group.updated` deletions to clean up grants), Agents Orchestrator (consume `group_membership.updated` to repatch live workload identities).

- **Identity service unchanged** — the [Identity](../architecture/identity.md) service gains a single documentation note explaining that Groups live in a separate service. No code change.

- **Egress Gateway unchanged** — v1 leaves PrivateResource and EgressRule as independent primitives. Operators pick one or the other per destination. Cross-primitive composition (injection on private resources) is deferred — tracked in [Open Questions](../open-questions.md).

- **Console** — the UI specified in [Product — Private Networks](../product/private-networks/private-networks.md) does not exist. Required screens: Network list, Network detail (with Tunnels tab showing credentials + liveness, Resources tab, and grants management on each resource), Group list, Group detail (with mixed-type member picker). The Console sidebar gains a "Private Networks" entry and a "Groups" entry under Org Settings.

- **Terraform provider** — `agyn_network`, `agyn_tunnel_credential`, `agyn_private_resource`, `agyn_private_resource_access`, `agyn_group`, `agyn_group_membership` resources do not exist in `agynio/terraform-provider-agyn`. Must be added with plan-time validation (reserved-zone checks on `intercept_host`, port cardinality match between `target_ports` and `intercept_ports`, principal-org membership cross-check). `agyn_tunnel_credential` returns the enrollment JWT as a sensitive output, retrievable once.

- **`agyn` CLI** — no commands exist. Add `agyn network {create,list,get,update,delete}`, `agyn tunnel {issue,list,revoke}`, `agyn resource {create,list,get,update,delete}`, `agyn resource grant {add,remove,list}`, `agyn group {create,list,get,update,delete}`, `agyn group member {add,remove,list}`.

## Acceptance Signal

- An organization owner creates a `Network` via the Console, the `agyn` CLI, and the Terraform provider — all three paths succeed and produce equivalent records.
- The owner issues a `TunnelCredential` for the network. The Console / CLI / Terraform provider returns the enrollment JWT once.
- The owner installs `ziti-edge-tunnel` on a host inside their private network, enrolls with the JWT, and the credential's status transitions to `Online` in the Console within 60 seconds.
- The owner creates a `PrivateResource` (`name: prod-postgres`, `protocol: tcp`, `host: 192.168.1.50`, `target_ports: ["5432"]`, `intercept_host: prod-postgres.internal.corp`, `intercept_ports: ["5432"]`) via all three entry points.
- The owner adds a `PrivateResourceAccess` grant for an agent. Within the documented 15-second window, an agent workload of that agent runs `psql -h prod-postgres.internal.corp -p 5432 -U app prod` and successfully connects to the Postgres on `192.168.1.50` through the tunneler.
- The owner adds a second `PrivateResourceAccess` grant for a `user` principal. The user enrolls a device via the Console; from that device they run the same `psql` command and successfully connect.
- The owner creates a `Group` and adds the agent + the user as members. The owner replaces the per-principal grants with a single group grant. Both principals continue to dial successfully; revoking the group grant immediately revokes both.
- A `PrivateResource` create attempt with `intercept_host: "*.ziti"` (or `*.svc`, `*.cluster.local`, or any pattern overlapping `100.64.0.0/10`) is rejected at create time with a clear validation error.
- A second `PrivateResource` create attempt with the same `(organization_id, intercept_host, intercept_port)` as an existing resource is rejected with a clear uniqueness error.
- The owner deletes the `Network`. All tunnel credentials, resources, and access grants in the Network are removed; the tunneler loses its session immediately; the resources are no longer dialable.
- The owner creates a second `TunnelCredential` in the same Network and enrolls a second tunneler on a different host. With both tunnels online, an agent dial succeeds via either; bringing one tunnel offline (`docker stop`) does not interrupt new connections to the resource — OpenZiti routes through the surviving tunnel.
- A `Group` add-member attempt with `member_type: runner` is rejected at create time.
- A `Group` is deleted while having outstanding `PrivateResourceAccess` grants for `group:<id>`. The grants are cascaded-deleted within the propagation window; agents that had access via the group lose it.
- An agent workload created after the agent is added to a group reflects the membership in its OpenZiti identity's role attributes (visible via `GET /edge/management/v1/identities/<id>` on the Controller). An agent workload running at the time the membership is changed has its live identity patched and the change reflects within the propagation window.
- A user enrolls a new device after being added to a group. The device's OpenZiti identity carries the `group-<id>` attribute and immediately resolves all group-granted resource accesses.
- Tunnel liveness transitions emit `tunnel_status.changed` notifications visible to subscribers (the Console updates within the SDK's push window).
- Reconciliation: manually deleting an OpenZiti service or Dial policy via the Controller API while the corresponding `PrivateResource` / `PrivateResourceAccess` row exists is repaired by the next reconciliation pass (default 60s). Manually creating an orphan service tagged `network-resources-<id>` with no matching row is removed by the same pass.

## Notes

- The PrivateResource and EgressRule primitives are independent in v1. Operators pick one or the other per destination; an EgressRule creating a Ziti service whose intercept overlaps an existing PrivateResource's intercept is rejected by the OpenZiti Controller with whatever error it returns (no friendly cross-primitive validation in v1).
- Injection on private resources (the "private GitLab + per-agent token" workflow) is not supported. Operators wanting per-agent credentials to private hosts wire them through agent ENVs via the [Secrets](../architecture/secrets.md) service. Tracked in [Open Questions](../open-questions.md).
- Per-agent injection variance for the same external domain (different agents in one org needing different tokens for the same destination) remains a known limitation of the existing EgressRule model and is not solved by this work. The fix (moving OpenZiti service ownership from per-rule to per-`(org, domain)`) is tracked separately in [Open Questions](../open-questions.md).
- The tunneler is unmodified open-source software. The platform's only contribution is the enrollment JWT and the role-attribute scheme. We do not ship a tunneler binary, do not extend the OpenZiti tunneler with custom logic, and do not push configuration to the tunneler over a side channel — the standard OpenZiti Controller-driven service-discovery mechanism is sufficient.
- Network is the HA boundary and the OpenZiti binding unit. In v1, single-tunnel-per-network is the common case; multi-tunnel HA works the same way (every tunnel in the network shares the `network-<id>` role attribute, all bind the same set of services, OpenZiti load-balances and fails over transparently). The data model supports HA without migration.
- Group nesting (`group#member` recursion on the `group` type itself) is intentionally omitted in v1. The OpenFGA model and resource shape are forward-compatible — adding nesting later is a model-only change. Tracked in [Open Questions](../open-questions.md).
- SCIM v2 endpoints (`/scim/v2/Users`, `/scim/v2/Groups`) are not part of v1. `Group.source` and `GroupMembership.source` exist now so SCIM can land later without a data model migration. Tracked in [Open Questions](../open-questions.md).
- Apps are not eligible access principals on PrivateResources in v1 (only `agent`, `user`, `group`). They are eligible group members. The OpenZiti pattern would be identical if extended; only the validation surface needs the Apps service org-membership check. Tracked in [Open Questions](../open-questions.md).
- Tracing and metering for traffic through PrivateResources are not part of v1. OpenZiti circuit metadata at the edge router is captured natively but not exported as platform observability. Tracked in [Open Questions](../open-questions.md).
- Agent-side resource discovery (an environment variable enumerating accessible private resources, or a platform API) is intentionally not provided. Agents dial by hostname blindly — the SDK only sees what it has policy for. If a discovery primitive becomes useful later, it can be added without changing the routing model. Tracked in [Open Questions](../open-questions.md).
