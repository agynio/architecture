# Egress Rules Address Private Resources

## Target

- [Product — Egress Gateway — Rule shape](../product/egress-gateway/egress-gateway.md#rule-shape)
- [Product — Egress Gateway — Attaching a private-destination rule grants access](../product/egress-gateway/egress-gateway.md#attaching-a-private-destination-rule-grants-access)
- [Product — Private Networks — EgressRule interaction](../product/private-networks/private-networks.md#egressrule-interaction)
- [Product — Console — Egress Rules](../product/console/console.md#egress-rules)
- [Product — Console — Private Resources](../product/console/console.md#private-resources)
- [Private Networks — EgressRule Interaction](../architecture/private-networks.md#egressrule-interaction)
- [Private Networks — Gateway Mediation](../architecture/private-networks.md#gateway-mediation)
- [EgressRules Service](../architecture/egress-rules-service.md)
- [Networks Service](../architecture/networks-service.md)
- [Egress Gateway — Private Resource Targets](../architecture/egress-gateway.md#private-resource-targets)
- [Resource Definitions — Egress Rule](../architecture/resource-definitions.md#egress-rule)
- [Resource Definitions — Private Resource](../architecture/resource-definitions.md#private-resource)
- [Authorization — EgressRules Service](../architecture/authz.md#egressrules-service)

## Delta

**Credentials cannot be injected into traffic bound for a private host.** An `EgressRule` matches only a public `domain_pattern`; a `PrivateResource` carries no policy at all. An operator who wants an agent to reach an internal GitLab with a token has to hand the token to the agent through its ENVs — the exact exposure the egress gateway exists to remove — and the two features collide rather than compose: a rule and a resource claiming the same hostname fail at the OpenZiti Controller with whatever error it happens to return.

`EgressRule.matcher` gains `private_resource_id`, mutually exclusive with `domain_pattern`. One is required; which one is set is what makes a rule public or private, and it is fixed for the rule's life. Everything else about the rule is unchanged — `methods`, `path_pattern`, `effect.action`, `effect.inject`, attachments to agents and environments, ≤15s propagation. `ports` stays public-only, since a private destination's ports are the resource's own.

Only `http` and `https` resources are eligible. A `tcp` resource is an opaque byte stream — a postgres or ssh session carries no headers to inject and no method or path to match — so there is nothing for a rule to act on; credentials for one still travel through the agent's ENVs.

Rules are not unique per destination, so several may name one resource. Two rules on one internal host, attached to different agents, is how each reaches it with its own token; a request matching more than one combines their effects under the existing merge semantics. Private targets are the clean case for this: a private-target rule provisions no interception of its own, so any number of them share the resource's single OpenZiti service without the ambiguity two public rules on one domain still have.

**Attaching a private-destination rule grants the target access to the resource.** For a public destination the agent could already reach the host and the rule only shapes the request; for a private one there is nothing to shape until the agent may connect, so the attachment carries both. It escalates nothing: `CreatePrivateResourceAccess` for an agent principal and `CreateEgressRuleAttachment` to that agent are authorized by the identical check — `can_edit_config` on the agent plus the cross-org guard — and the same holds for environments. Grants stay the only path for `user`, `app`, and `group` principals and for `tcp` resources; a resource's effective principal set is the union of both, surfaced in one list on the resource.

### Supporting changes

**Gateway mediation.** A `PrivateResource` gains a derived `mediation` state (`tunnel` | `egress_gateway`), `egress_gateway` exactly while some rule names it. Mediated, its front service `private-<id>` is rebound from the network's Tunnels to the Egress Gateway and given a forwarding `host.v1`, and one tunnel-bound `private-<id>-upstream-<intercept_port>` service per intercept port is created for the gateway to dial. The [Networks service](../architecture/networks-service.md) owns every one of those writes and exposes `SetPrivateResourceMediation` for the [EgressRules service](../architecture/egress-rules-service.md) to call on the resource's first rule and last. Routing *every* `http`/`https` resource through the gateway unconditionally was rejected: it makes the gateway an availability dependency for private traffic no operator asked to inspect.

**The intercept→target port mapping lives in the upstream services, not in a forwarded port.** The gateway sees only the intercept side of a connection, and intercept and target ports differ routinely (`443` → `8443`), so forwarding the dialed port to the tunnel would reach the wrong port — or be refused by an `allowedPortRanges` that lists target ports. One upstream service per intercept port, each naming its target port statically, makes choosing the service the act of applying the mapping. Neither the gateway nor the tunnel carries a mapping table.

**Callers without a rule are spliced, not intercepted.** Mediation is a property of the service, so every dialer of a mediated resource lands on the gateway — including principals holding only an access grant, whose clients have no reason to trust the Egress CA. The gateway resolves the dialer identity before the TLS handshake and, when no attached rule names the resource, splices the connection to the upstream service byte-for-byte: nothing decrypted, no substituted certificate. Without this, one agent's rule would break every user device dialing that resource.

**Upstream TLS is configurable for private destinations.** A rule gains `upstream_tls` (`server_name`, `ca_bundle_secret_id` xor `insecure_skip_verify`), used on the gateway→target leg of an `https` resource. Internal endpoints routinely present a corporate-CA or self-signed certificate, and `intercept_host` need not be the name on it. Unset, verification is identical to a public destination — the correct default, and a loud failure rather than a silent trust.

**Cached resource fields need their own invalidation.** The gateway's rule cache has no TTL and was invalidated only by rule-scoped events, but a private-target rule carries the resource's `intercept_host` and `protocol` denormalized — and `UpdatePrivateResource` can change both without touching a rule. The gateway now also consumes `private_resource.updated` from the organization's Notifications room, the same mechanism it already uses for rule changes. Routing never depends on the cached values: the destination comes from the OpenZiti service the connection arrived on and the upstream from `(resource_id, intercept port)`, both read live.

**A rule set that cannot be determined refuses the connection.** On a cache miss while the EgressRules service is unreachable, the gateway cannot tell "no rule applies" from "cannot tell" — and for a mediated resource those differ by whether a `deny` is enforced and a credential injected. Guessing *splice* would send an unauthenticated request to an internal host and log nothing. Public destinations are unaffected: one the gateway holds no rules for is one the sidecar would never have routed to it.

**Referential integrity in both directions.** The two services now depend on each other, request-scoped and off any startup path: EgressRules calls `GetPrivateResource`, `SetPrivateResourceMediation`, and `ListPrivateResourcesReachableBy`; Networks calls `CountRulesReferencingPrivateResource` (delete and protocol-change guards), `ListMediatedPrivateResources` (one call per organization per reconciliation pass), and `ListAttachedRuleDomains`. Both write Dial policies against `@private-<resource_id>` and neither disturbs the other's, because each sweeps only what carries its own `agyn.managed_by` tag.

**Hostname collisions are detected, not prevented.** An intercept is ambiguous only for an identity authorized to dial both sides, so an organization may hold a public rule for `gitlab.corp` and a resource intercepting `gitlab.corp` at once — nothing is wrong until one workload can dial both. Checking at create time would reject configurations that never collide; checking at attach and grant time catches the direct orderings but cannot be an invariant, because the authorization is assembled by five write paths across four services. A grant to a group collides with a rule attached to a member, and the membership that completes it is a Groups operation. A rule on an environment collides with a grant to an agent running it, and repointing that agent is an Agents operation. Both services would need a cross-service overlap query in the middle of operations that have nothing to do with networking, and would still race concurrent writes.

So the two direct checks stay as a fast-fail — expanded through groups and environments, and labelled best-effort — and each reconciliation pass reports every identity authorized for two overlapping interceptions, on both the rule and the resource. Nothing is auto-resolved: which side the operator meant is not inferable, and revoking either would cut off working traffic.

**No `provisioning_state` is added to `EgressRule`.** Reconciliation here repairs rather than records — a rule's OpenZiti materialization is re-derivable from the row on every pass, unlike the Networks resources whose failed provisioning must survive to be retried. Conditions the reconciler cannot repair surface as logs, metrics, and Console warnings instead.

## Acceptance Signal

- An operator creates an egress rule with an `https` private resource as its destination, injects a secret-backed `Authorization` header, and attaches it to an agent. The agent `curl`s the resource's hostname and the internal host receives the credential; the agent's container never held it.
- The same agent, before the attachment, cannot reach the resource at all.
- Detaching the rule stops both the injection and the access within ≤15s.
- A rule naming a `tcp` resource is refused at create time, as is one whose private resource belongs to another organization.
- A second rule on the same resource, attached to a different agent, injects a different token; each agent reaches the host with its own.
- A resource intercepting `443` onto target port `8443` is reachable through a rule — the gateway dials the upstream service for intercept port `443` and the tunnel connects to `8443`. A resource with `[80, 443] → [8080, 8443]` works on both ports, each landing on its pair.
- A user with an access grant on the same resource, dialing from an enrolled device while the agent's rule exists, completes TLS against the **target's own certificate** — not one signed by the Egress CA — and reaches the host.
- Creating that first rule reset live connections to the resource; deleting the last rule resets them again and returns the resource to the direct tunnel path.
- An `https` resource behind a corporate-CA certificate fails with a TLS error until `upstream_tls.ca_bundle_secret_id` or `server_name` is set, then succeeds.
- Renaming a mediated resource's `intercept_host` is reflected in the gateway's spans on the next request, without a gateway restart.
- With the EgressRules service stopped and the gateway's cache cold, a dial to a mediated resource is refused rather than passed through uninspected.
- Deleting the resource is refused while any rule names it, listing them; so is changing its protocol to `tcp`.
- A public rule for `gitlab.corp` and a resource intercepting `gitlab.corp` both create successfully. Attaching that rule to an agent already granted the resource is refused with both named, as is granting the resource to an agent that already has the rule attached — including when the grant reached the agent through a group or its environment.
- Reaching the same state by a path no write-time check sees produces a reconciliation warning on both the rule and the resource within one pass, naming the affected identity and how each side was acquired. Each of these does it: adding an agent to a group that holds the grant; repointing an agent at an environment the rule is attached to; granting the resource to a group whose member holds the rule.
- The resource detail page's access list shows the rule-attached agent and a directly-granted user side by side, each labelled with its source.
- Killing the Networks service mid-flip leaves the resource in a partial state that the next reconciliation pass repairs without operator action.
- Requests to a rule-covered private resource emit `egress.request` spans carrying `egress.private_resource_id`; spliced connections emit spans with `egress.outcome: spliced` and no method or path.

## Notes

- **The gateway learns nothing new about private networks.** It identifies the destination from the OpenZiti service the connection arrived on and derives the upstream service name from that resource ID and the intercept port `forwardPort` gave it — no target host, no target port, no Networks-owned identifier. Only `intercept_host` and `protocol` ride along in the rule-lookup response, for span labels and TLS expectation.
- **Two public rules on one domain pattern are still ambiguous at the interception layer.** Removing the uniqueness constraint (#184) made the record layer allow what per-rule OpenZiti services do not. Private targets do not inherit the problem — they provision no interception — and the public half is tracked in [Conditions List per Rule](../open-questions.md#egress-gateway-conditions-list-per-rule).
- **Rules attach to agents and environments only.** Users, apps, and groups reach private resources exclusively through access grants — they run no HTTP-shaped workload the platform can inject into.
- **The Terraform provider and `agyn` CLI surfaces are not specified here.** Both expose egress rule CRUD and will need the destination selection; that is their own spec work.
