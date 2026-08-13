# Private Networks

## Purpose

Private Networks let operators give agents access to resources inside their own private networks — a home lab, a corporate VPC, an on-prem datacenter — without exposing those resources to the public internet. The operator runs a standard OpenZiti tunneler inside their network; the tunneler enrolls into the platform and bridges traffic to declared resources. Agents granted access dial those resources by hostname, the same way they dial any other endpoint, with no special tooling and no platform-side knowledge of the private host's address.

Three capabilities:

- **Reach private hosts** — agents can dial `prod-postgres.internal.corp:5432`, `gitlab.lan:443`, or any other endpoint inside the operator's network without that endpoint being publicly routable.
- **Per-principal access control** — each resource has an explicit access list. Agents, environments (which covers sandboxes), individual users (via their enrolled devices), and groups can be granted dial access; revocation is immediate.
- **HA via multiple tunnels** — a Network can have multiple tunneler instances installed in different locations. OpenZiti load-balances and fails over between them transparently.

The agent runs unmodified tools (`psql`, `curl`, `git`, language-specific HTTP clients). It connects to a hostname; the platform routes the connection through the tunneler to the real host.

## User Stories

- As an operator, I want my agent to query an internal PostgreSQL inside my VPC without opening the database to the public internet or running a bastion.
- As an operator, I want my agent to clone from a private GitLab on my office LAN by dialing its real hostname, with no special URL rewriting.
- As an operator, I want to grant a team of analysts SSH access to internal hosts from their own machines, using their device identities, with permissions managed by group membership.
- As an engineer at a sandbox shell, I want to reach the internal database my agents already reach, without an operator having to invent an agent for me to borrow.
- As an operator, I want to run two tunneler instances in different availability zones of my VPC so a single host outage doesn't take down access.
- As an operator, I want to revoke an agent's access to a private database the moment I detect anomalous behavior — without restarting the agent.

## Concepts

| Term | Definition |
|---|---|
| **Network** | A named container for a private network's resources and the tunneler instances that reach it. Networks are organization-scoped. A Network has no settings beyond a name and description — its purpose is to be the OpenZiti binding boundary and the unit of HA. |
| **Tunnel credential** | An enrollment artifact issued by the platform for a single tunneler instance. The credential is a one-time-token JWT plus an install snippet for the supported tunneler distributions (Docker, Linux/macOS binary, Kubernetes helm chart). One Tunnel belongs to exactly one Network; a Network can have many Tunnels. |
| **Private resource** | A single addressable endpoint behind the Network — a `target_host:target_ports` target the Tunnel forwards to, exposed to agents as a hostname they dial. A resource has a single protocol (`tcp`, `http`, or `https`). UDP is not supported in v1. |
| **Access** | The list of principals (agents, environments, users, apps, groups) authorized to dial a Private Resource. A user with access dials from their enrolled devices; an app dials via its own identity; a group with access grants every member transitively; an environment grants every workload running it, including sandboxes. An agent or environment can also be granted access by attaching an [EgressRule](#egressrule-interaction) that names the resource; the list shows both. |
| **Group** | An org-scoped named collection of identities (users, agents, apps). Granting access to a group is equivalent to granting to every member. See [Groups](../../architecture/groups-service.md). |

## How traffic flows

Without any [EgressRule](../../architecture/egress-rules-service.md) naming the resource:

```
Agent → resource hostname → OpenZiti edge router → Tunneler → real host:port
```

The agent's connection is intercepted by the Ziti sidecar in its pod, routed over OpenZiti to the tunneler the operator is running, and forwarded to the real target. The agent's code observes a normal TCP connection.

For TCP resources (postgres, ssh, redis, raw protocols) this is always the path. Header injection is not applicable.

Once a rule names an HTTP/HTTPS resource, the Egress Gateway is inserted into the path:

```
Agent → resource hostname → edge router → Egress Gateway → edge router → Tunneler → real host:port
```

The agent dials the same hostname on the same port and observes the same normal connection — see [EgressRule Interaction](#egressrule-interaction).

## Adding a tunnel

1. **Create a Network** in the Console (Networking → Private Networks → New). Pick a name. No other configuration is required.
2. **Issue a Tunnel Credential** for the network. The Console shows the enrollment JWT once and offers install snippets for the supported tunneler distributions.
3. **Run the tunneler** on a host inside the private network. The host must have outbound connectivity to the OpenZiti Controller and the edge routers (standard requirement of any OpenZiti tunneler). It does **not** require inbound connectivity, public IP, or open ports.
4. The tunneler's status flips to **Online** in the Console within seconds of enrollment.

For HA, issue multiple credentials in the same Network and run each on a separate host. All tunnels in a Network share the same set of bindable resources; OpenZiti picks one transparently and fails over if a tunnel goes offline.

### Console layout

Networks and resources are **two sidebar sections, not one nested surface**. Both live in the Console's **Networking** group — see [Console — Organization Sections](../console/console.md#organization-sections):

| Section | Contains |
|---|---|
| **Private Networks** | The network list, and per network its name, description, and **tunnels**. Nothing else — a network has no settings beyond name and description, so there is nothing to tab between |
| **Private Resources** | Every resource in the organization, across all networks. Each has its own detail page owning the resource's fields and its access grants |

Resources are listed at **organization** scope, not under their network. That matches the constraint the model already enforces: `intercept_host` + port uniqueness is checked across the whole organization (see [Uniqueness](../../architecture/private-networks.md#privateresource)). A per-network list would present a namespace that does not exist, and could not explain a cross-network collision to the operator who hit it.

`Private Resources` sits directly beside `Egress Rules` in the group. That adjacency is deliberate: an egress rule can name a private resource as its destination, and the two lists are read together — see [EgressRule interaction](#egressrule-interaction).

Groups live under **Organization → Groups**, but are also reachable as an inline "Create group" affordance in the resource access picker — same data, two entry points.

Each resource detail page surfaces a **Copy connection string** affordance (`prod-postgres.corp:5432`) for fast paste-into-agent-config or paste-into-tooling workflows. There is no agent-side discovery API in v1 — operators configure agents with hostnames through the agent's system prompt, skills, or external runbooks.

## Adding a private resource

Each resource declares a target (what the tunnel forwards to) and an intercept (what agents dial):

| Field | Description |
|---|---|
| **Name** | A human-readable label (e.g., `prod-postgres`). Not unique — names are for operators, IDs are for routing |
| **Protocol** | `tcp`, `http`, or `https`. Determines whether an [EgressRule](#egressrule-interaction) can name the resource later — only `http` and `https` can. UDP is not supported in v1 |
| **Target host** | IP literal or DNS name the tunneler forwards to. Resolved at the tunnel side at connect time — internal DNS names (`prod-db.internal.corp`, `*.us-west-2.rds.amazonaws.com`) work because the tunneler sits inside the operator's network |
| **Target ports** | List of port numbers on the target host (e.g., `5432`, or `[80, 443]`, or `[9200, 9300]`). Port ranges are not supported in v1 |
| **Intercept host** | Hostname the agent dials. The operator picks this freely — it can match the real internal hostname (`gitlab.internal.corp`), or use a synthetic platform-side name |
| **Intercept ports** | Ports the agent dials. Cardinality and order must match Target ports; the mapping is positional 1:1 |

Reserved zones (`*.agyn`, `*.svc`, `*.cluster.local`, OpenZiti's synthetic CIDR, `localhost`, `127.0.0.0/8`, `::1/128`) are rejected to avoid collision with platform routing.

If an operator picks a real public hostname (e.g., `gitlab.com`) as the intercept, **all** agent traffic to that hostname is routed through the tunnel, including legitimate public traffic. The Console warns at create time but does not block — this is an operator choice.

A resource is dialable as soon as it is created and at least one tunneler in its Network is online — but it has no agents with access until the operator adds grants.

## Granting access

Access is managed on the resource detail page or via the resource's access list. Each grant binds a principal to the resource:

| Principal | Effect |
|---|---|
| **Agent** | The agent's workloads can dial the resource |
| **Environment** | Every workload running the environment can dial — agents pointed at it and [sandboxes](../sandboxes/sandboxes.md) started in it. The only way to give a sandbox access |
| **User** | The user's enrolled devices can dial the resource. Useful for human-driven workflows — a dev's laptop can `psql` an internal DB through the same network |
| **App** | The app dials the resource using its own OpenZiti identity. Useful for platform-installed apps that need direct access to a private service |
| **Group** | Every group member (users, agents, apps) can dial. Membership changes propagate automatically |

**Granting to an environment is a broader act than granting to an agent.** A sandbox is a shell, and anyone who may start one in that environment gets the resource with it — so the grant follows the environment's edit permission, the same one that governs its secrets and egress rules. Grant to the agent when one agent needs the resource; grant to the environment when the people and workloads *working in* that environment do.

Granting to an individual agent instance or to one specific sandbox is not supported. Both are narrower than an environment, and neither survives a workload restart today.

An agent or environment can also get access by having an [egress rule](#egressrule-interaction) for the resource attached to it — the path to take when it needs injected credentials as well as reachability. The access list shows those principals alongside the grants, labelled with where the access came from.

Revocation deletes the underlying OpenZiti dial policy immediately. In-flight connections that depended on the policy are torn down. **Propagation window** to live workloads / SDKs: ≤15 seconds (dominated by the SDK's service-list poll interval, the same propagation behavior the [Egress Gateway](../egress-gateway/egress-gateway.md#attaching-rules-to-agents) uses).

## EgressRule interaction

PrivateResource and [EgressRule](../egress-gateway/egress-gateway.md) describe different halves of one destination. The resource says where the endpoint is and how to reach it; a rule naming that resource says what happens to HTTP requests going there.

| Need | Use |
|---|---|
| Direct TCP, HTTP, or HTTPS access to a private host, no header injection | PrivateResource + access grant |
| Header injection or deny rules on an external hostname | EgressRule with a public destination |
| Header injection or deny rules on a private host (e.g., a token to an internal GitLab) | EgressRule with that PrivateResource as its destination |
| Any of the above for a user's device, an app, or a group | PrivateResource + access grant — rules attach to agents and environments only |

Creating a rule for a resource is what puts the platform's [Egress Gateway](../egress-gateway/egress-gateway.md) in front of it. Only `http` and `https` resources qualify: a `tcp` resource is an opaque byte stream — a postgres or ssh session has no headers to inject and no method or path to match — so credentials for one still travel through the agent's ENVs.

A resource can carry several rules. Two rules on one internal GitLab, attached to different agents, is how each agent reaches it with its own token.

### Two ways to grant access

**Attaching a rule to an agent grants that agent access to the resource**, alongside applying the rule. Detaching takes it back. That is deliberate: a credential is useful only to someone who can connect, and asking the operator to say so twice is what made the two features feel separate.

| Path | Principals | Grants access | Injects / denies |
|---|---|---|---|
| Access grant on the resource | agent, environment, user, app, group | ✓ | — |
| Egress rule attachment | agent, environment | ✓ | ✓ |

Neither path opens more than the other: adding an agent to a resource's access list and attaching a rule to that agent require the same permission on the agent, and the same holds for environments. A principal may hold both; each is revoked where it was created.

**The resource's access list remains the answer to "who can reach this?"** — it lists principals granted directly and principals reaching the resource through an attached rule, each labelled with where it came from.

### What changes for callers when a rule exists

The first rule on a resource moves its traffic onto the gateway, and deleting the last one moves it back. Two consequences worth knowing:

- **The flip resets live connections to that resource**, for every caller — the same interruption as detaching a rule, within the same ≤15s window.
- **Callers with no rule attached are unaffected otherwise.** Their traffic passes through the platform untouched: not decrypted, no substituted certificate. An analyst dialing an internal HTTPS host from their laptop sees the target's real certificate whether or not some agent has a rule on it.

## Observability

Tunnel liveness is surfaced per credential in the Console: `Online`, `Offline`, `Never enrolled`, with last-seen timestamps sourced from the OpenZiti Controller's session info. A Network's overall reachability is derived: a resource is **reachable** if at least one of its Network's tunnels is online, **degraded** otherwise.

Per-request tracing and metering arrive with a rule: traffic the [Egress Gateway](../egress-gateway/egress-gateway.md#observability) inspects emits the same spans and metering records as any other egress traffic. Resources with no rule on them, and every `tcp` resource, are still unobserved at the request level — see [open-questions](../../open-questions.md).

## Lifecycle

Resources are created, edited, and deleted by organization owners through the Console, the `agyn` CLI, or the Terraform provider.

| Event | Effect |
|---|---|
| Network created | No effect on traffic until tunnels are enrolled and resources are added |
| Tunnel credential issued | JWT shown once; not retrievable afterward. The tunneler must enroll within the JWT's validity window (typically 24h) |
| Tunneler enrolls | Tunnel goes Online within seconds; resources in the Network become reachable through it |
| Tunneler offline | Resources fall back to remaining tunnels in the Network; reachability depends on whether any tunnel is online |
| Resource created | Becomes dialable once any tunnel in the Network is online and at least one access grant exists |
| Access grant added | Within ≤15s, the granted principal can dial the resource |
| Access grant revoked | Within ≤15s, the principal can no longer dial. In-flight connections reset |
| First egress rule created for the resource | Its traffic moves onto the [Egress Gateway](../egress-gateway/egress-gateway.md). Live connections reset, for every caller |
| Last egress rule on the resource deleted | Its traffic moves back to the direct tunnel path. Live connections reset again |
| Tunnel credential revoked | The tunneler holding it loses its session immediately. Other tunnels in the Network are unaffected |
| Resource deleted | Refused while any egress rule names it. Otherwise all access grants on the resource are cascaded-deleted and in-flight connections reset |
| Network deleted | All tunnel credentials, resources, and access grants in the Network are cascaded-deleted |
| Group membership changes | All resources granted to that group reflect the new membership within the propagation window |

## Constraints

- A resource has a single protocol (`tcp`, `http`, or `https`). UDP is not supported in v1. Mixed-protocol targets on the same host = multiple resources.
- A resource may declare multiple ports as a list (e.g., `[5432]`, `[80, 443]`, `[9200, 9300]`), but port ranges are not supported in v1. The intercept-to-target port mapping is positional 1:1 and the protocol is shared.
- Reserved zones are rejected on `intercept_host`: `*.agyn`, `*.svc`, `*.cluster.local`, any pattern overlapping `100.64.0.0/10`, `localhost`, `127.0.0.0/8`, and `::1/128`.
- For each port in `intercept_ports`, the tuple `(intercept_host, port)` must be unique across all resources in the organization. Operators namespace by hostname (`prod-postgres.corp:5432` vs `dev-postgres.corp:5432`).
- A Tunnel belongs to one Network. Running one tunneler for two networks requires two separate tunneler installations.
- Runners are not eligible group members and not eligible access principals — they are infrastructure, not actors in the access model.
- Environments are not group members. A group collects identities; an environment is a configuration resource, and it is a principal here only because it is what an agent workload and a sandbox have in common.
- An `intercept_host` may equal some egress rule's public domain pattern. What breaks is one agent holding both, since its dials to that hostname become ambiguous — the fix is a rule with the resource as its destination. Granting or attaching the second one directly is refused; arriving at the same state through group membership or a change of environment is not, and is reported by reconciliation instead.
- Deleting a resource, or changing its protocol to `tcp`, is refused while any egress rule names it.

## Related Architecture

- [Private Networks](../../architecture/private-networks.md) — architecture overview
- [Networks Service](../../architecture/networks-service.md) — control-plane CRUD, reconciliation, OpenZiti resource lifecycle
- [Groups Service](../../architecture/groups-service.md) — Group / GroupMembership lifecycle
- [Resource Definitions](../../architecture/resource-definitions.md) — canonical schemas for Network, TunnelCredential, PrivateResource, PrivateResourceAccess, Group, GroupMembership
- [OpenZiti Integration](../../architecture/openziti.md) — overlay infrastructure, tunnel identity lifecycle, group role-attribute sync
- [Egress Gateway](../egress-gateway/egress-gateway.md) — the policy layer over outbound HTTP/HTTPS, to public destinations and to the resources here
- [Authorization](../../architecture/authz.md) — OpenFGA `group` type and per-service authorization checks
