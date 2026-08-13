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

`EgressRule.matcher` gains `private_resource_id`, mutually exclusive with `domain_pattern`. One is required; which one is set is what makes a rule public or private, and it is fixed for the rule's life. Everything else about the rule is unchanged — `methods`, `path_pattern`, `effect.action`, `effect.inject`, attachments to agents and environments, ≤15s propagation. Only `http` and `https` resources are eligible; `ports` stays public-only, since a private destination's ports are the resource's own.

**Attaching a private-destination rule grants the target access to the resource.** For a public destination the agent could already reach the host and the rule only shapes the request; for a private one there is nothing to shape until the agent may connect, so the attachment carries both. It escalates nothing: `CreatePrivateResourceAccess` for an agent principal and `CreateEgressRuleAttachment` to that agent are authorized by the identical check — `can_edit_config` on the agent plus the cross-org guard — and the same holds for environments. Grants stay the only path for `user`, `app`, and `group` principals and for `tcp` resources; a resource's effective principal set is the union of both, surfaced in one list on the resource.

### Supporting changes

**Gateway mediation.** A `PrivateResource` gains a derived `mediation` state (`tunnel` | `egress_gateway`), `egress_gateway` exactly while some rule names it. Mediated, its front service `private-<id>` is rebound from the network's Tunnels to the Egress Gateway and given a forwarding `host.v1`, and a second service `private-<id>-upstream` — tunnel-bound, carrying the real target — is created for the gateway to dial. The [Networks service](../architecture/networks-service.md) owns every one of those writes and exposes `SetPrivateResourceMediation` for the [EgressRules service](../architecture/egress-rules-service.md) to call on the resource's first rule and last. Routing *every* `http`/`https` resource through the gateway unconditionally was rejected: it makes the gateway an availability dependency for private traffic no operator asked to inspect.

**Callers without a rule are spliced, not intercepted.** Mediation is a property of the service, so every dialer of a mediated resource lands on the gateway — including principals holding only an access grant, whose clients have no reason to trust the Egress CA. The gateway resolves the dialer identity before the TLS handshake and, when no attached rule names the resource, splices the connection to the upstream service byte-for-byte: nothing decrypted, no substituted certificate. Without this, one agent's rule would break every user device dialing that resource.

**Upstream TLS is configurable for private destinations.** A rule gains `upstream_tls` (`server_name`, `ca_bundle_secret_id` xor `insecure_skip_verify`), used on the gateway→target leg of an `https` resource. Internal endpoints routinely present a corporate-CA or self-signed certificate, and `intercept_host` need not be the name on it. Unset, verification is identical to a public destination — the correct default, and a loud failure rather than a silent trust.

**Referential integrity in both directions.** The two services now depend on each other, request-scoped and off any startup path: EgressRules calls `GetPrivateResource` and `SetPrivateResourceMediation`; Networks calls `CountRulesReferencingPrivateResource` (delete and protocol-change guards) and `ListMediatedPrivateResources` (one call per organization per reconciliation pass). Both write Dial policies against `@private-<resource_id>` and neither disturbs the other's, because each sweeps only what carries its own `agyn.managed_by` tag.

**Hostname collisions become validation errors.** A `domain_pattern` equal to an existing `intercept_host` and an `intercept_host` equal to an existing `domain_pattern` are both rejected at create time, pointing at a private-destination rule instead of letting the Controller reject the second `CreateService`.

## Acceptance Signal

- An operator creates an egress rule with an `https` private resource as its destination, injects a secret-backed `Authorization` header, and attaches it to an agent. The agent `curl`s the resource's hostname and the internal host receives the credential; the agent's container never held it.
- The same agent, before the attachment, cannot reach the resource at all.
- Detaching the rule stops both the injection and the access within ≤15s.
- A rule naming a `tcp` resource is refused at create time. So is a second rule naming an already-ruled resource, and a rule whose private resource belongs to another organization.
- A user with an access grant on the same resource, dialing from an enrolled device while the agent's rule exists, completes TLS against the **target's own certificate** — not one signed by the Egress CA — and reaches the host.
- Creating that first rule reset live connections to the resource; deleting the last rule resets them again and returns the resource to the direct tunnel path.
- An `https` resource behind a corporate-CA certificate fails with a TLS error until `upstream_tls.ca_bundle_secret_id` or `server_name` is set, then succeeds.
- Deleting the resource is refused while the rule exists, naming it; so is changing its protocol to `tcp`.
- Creating a public rule whose `domain_pattern` equals an existing `intercept_host` is refused with a message pointing to a private destination, and vice versa.
- The resource detail page's access list shows the rule-attached agent and a directly-granted user side by side, each labelled with its source.
- Killing the Networks service mid-flip leaves the resource in a partial state that the next reconciliation pass repairs without operator action.
- Requests to a rule-covered private resource emit `egress.request` spans carrying `egress.private_resource_id`; spliced connections emit spans with `egress.outcome: spliced` and no method or path.

## Notes

- **The gateway learns nothing new about private networks.** It identifies the destination from the OpenZiti service the connection arrived on and dials `private-<resource_id>-upstream` by name. The resource's `intercept_host`, ports, and protocol ride along in the rule-lookup response, so the Networks service stays off the request path entirely.
- **One rule per resource means one credential per resource.** Two agents needing distinct tokens to the same internal host cannot be expressed; that constraint is inherited from public destinations and is now an [open question](../open-questions.md) in its own right.
- **Rules attach to agents and environments only.** Users, apps, and groups reach private resources exclusively through access grants — they run no HTTP-shaped workload the platform can inject into.
- **The Terraform provider and `agyn` CLI surfaces are not specified here.** Both expose egress rule CRUD and will need the destination selection; that is their own spec work.
