# Private Networks

## Purpose

Private Networks let operators give agents access to resources inside their own private networks — a home lab, a corporate VPC, an on-prem datacenter — without exposing those resources to the public internet. The operator runs a standard OpenZiti tunneler inside their network; the tunneler enrolls into the platform and bridges traffic to declared resources. Agents granted access dial those resources by hostname, the same way they dial any other endpoint, with no special tooling and no platform-side knowledge of the private host's address.

Three capabilities:

- **Reach private hosts** — agents can dial `prod-postgres.internal.corp:5432`, `gitlab.lan:443`, or any other endpoint inside the operator's network without that endpoint being publicly routable.
- **Per-principal access control** — each resource has an explicit access list. Agents, individual users (via their enrolled devices), and groups can be granted dial access; revocation is immediate.
- **HA via multiple tunnels** — a Network can have multiple tunneler instances installed in different locations. OpenZiti load-balances and fails over between them transparently.

The agent runs unmodified tools (`psql`, `curl`, `git`, language-specific HTTP clients). It connects to a hostname; the platform routes the connection through the tunneler to the real host.

## User Stories

- As an operator, I want my agent to query an internal PostgreSQL inside my VPC without opening the database to the public internet or running a bastion.
- As an operator, I want my agent to clone from a private GitLab on my office LAN by dialing its real hostname, with no special URL rewriting.
- As an operator, I want to grant a team of analysts SSH access to internal hosts from their own machines, using their device identities, with permissions managed by group membership.
- As an operator, I want to run two tunneler instances in different availability zones of my VPC so a single host outage doesn't take down access.
- As an operator, I want to revoke an agent's access to a private database the moment I detect anomalous behavior — without restarting the agent.

## Concepts

| Term | Definition |
|---|---|
| **Network** | A named container for a private network's resources and the tunneler instances that reach it. Networks are organization-scoped. A Network has no settings beyond a name and description — its purpose is to be the OpenZiti binding boundary and the unit of HA. |
| **Tunnel credential** | An enrollment artifact issued by the platform for a single tunneler instance. The credential is a one-time-token JWT plus an install snippet for the supported tunneler distributions (Docker, Linux/macOS binary, Kubernetes helm chart). One Tunnel belongs to exactly one Network; a Network can have many Tunnels. |
| **Private resource** | A single addressable endpoint behind the Network — a `target_host:target_ports` target the Tunnel forwards to, exposed to agents as a hostname they dial. A resource has a single protocol (`tcp`, `http`, or `https`). UDP is not supported in v1. |
| **Access** | The list of principals (agents, users, apps, groups) authorized to dial a Private Resource. A user with access dials from their enrolled devices; an app dials via its own identity; a group with access grants every member transitively. |
| **Group** | An org-scoped named collection of identities (users, agents, apps). Granting access to a group is equivalent to granting to every member. See [Groups](../../architecture/groups-service.md). |

## How traffic flows

Without injection (no [EgressRule](../../architecture/egress-rules-service.md) attached to the resource):

```
Agent → resource hostname → OpenZiti edge router → Tunneler → real host:port
```

The agent's connection is intercepted by the Ziti sidecar in its pod, routed over OpenZiti to the tunneler the operator is running, and forwarded to the real target. The agent's code observes a normal TCP connection.

For TCP resources (postgres, ssh, redis, raw protocols) this is always the path. Header injection is not applicable.

For HTTP/HTTPS resources with injection rules attached, the Egress Gateway is inserted into the path — see [EgressRule Interaction](#egressrule-interaction).

## Adding a tunnel

1. **Create a Network** in the Console (Org Settings → Private Networks → New). Pick a name. No other configuration is required.
2. **Issue a Tunnel Credential** for the network. The Console shows the enrollment JWT once and offers install snippets for the supported tunneler distributions.
3. **Run the tunneler** on a host inside the private network. The host must have outbound connectivity to the OpenZiti Controller and the edge routers (standard requirement of any OpenZiti tunneler). It does **not** require inbound connectivity, public IP, or open ports.
4. The tunneler's status flips to **Online** in the Console within seconds of enrollment.

For HA, issue multiple credentials in the same Network and run each on a separate host. All tunnels in a Network share the same set of bindable resources; OpenZiti picks one transparently and fails over if a tunnel goes offline.

### Console layout

Private Networks lives as a top-level item in the Console sidebar under the **Infrastructure** group (alongside Egress Gateway, Runners, etc.) — it's operational machinery operators set up once, not per-thread configuration. Groups live separately under **Org Settings → Groups**, but are also reachable as an inline "Create group" affordance in the resource access picker — same data, two entry points.

Each resource detail page surfaces a **Copy connection string** affordance (`prod-postgres.corp:5432`) for fast paste-into-agent-config or paste-into-tooling workflows. There is no agent-side discovery API in v1 — operators configure agents with hostnames through the agent's system prompt, skills, or external runbooks.

## Adding a private resource

Each resource declares a target (what the tunnel forwards to) and an intercept (what agents dial):

| Field | Description |
|---|---|
| **Name** | A human-readable label (e.g., `prod-postgres`). Not unique — names are for operators, IDs are for routing |
| **Protocol** | `tcp`, `http`, or `https`. Determines whether header injection via [EgressRule](../../architecture/egress-rules-service.md) is possible later. UDP is not supported in v1 |
| **Target host** | IP literal or DNS name the tunneler forwards to. Resolved at the tunnel side at connect time — internal DNS names (`prod-db.internal.corp`, `*.us-west-2.rds.amazonaws.com`) work because the tunneler sits inside the operator's network |
| **Target ports** | List of port numbers on the target host (e.g., `5432`, or `[80, 443]`, or `[9200, 9300]`). Port ranges are not supported in v1 |
| **Intercept host** | Hostname the agent dials. The operator picks this freely — it can match the real internal hostname (`gitlab.internal.corp`), or use a synthetic platform-side name |
| **Intercept ports** | Ports the agent dials. Cardinality and order must match Target ports; the mapping is positional 1:1 |

Reserved zones (`*.ziti`, `*.svc`, `*.cluster.local`, OpenZiti's synthetic CIDR, `localhost`, `127.0.0.0/8`, `::1/128`) are rejected to avoid collision with platform routing.

If an operator picks a real public hostname (e.g., `gitlab.com`) as the intercept, **all** agent traffic to that hostname is routed through the tunnel, including legitimate public traffic. The Console warns at create time but does not block — this is an operator choice.

A resource is dialable as soon as it is created and at least one tunneler in its Network is online — but it has no agents with access until the operator adds grants.

## Granting access

Access is managed on the resource detail page or via the resource's access list. Each grant binds a principal to the resource:

| Principal | Effect |
|---|---|
| **Agent** | The agent's workloads can dial the resource |
| **User** | The user's enrolled devices can dial the resource. Useful for human-driven workflows — a dev's laptop can `psql` an internal DB through the same network |
| **App** | The app dials the resource using its own OpenZiti identity. Useful for platform-installed apps that need direct access to a private service |
| **Group** | Every group member (users, agents, apps) can dial. Membership changes propagate automatically |

Revocation deletes the underlying OpenZiti dial policy immediately. In-flight connections that depended on the policy are torn down. **Propagation window** to live workloads / SDKs: ≤15 seconds (dominated by the SDK's service-list poll interval, the same propagation behavior the [Egress Gateway](../egress-gateway/egress-gateway.md#attaching-rules-to-agents) uses).

## EgressRule interaction

PrivateResource and [EgressRule](../egress-gateway/egress-gateway.md) are independent primitives. An operator picks one or the other per destination:

| Need | Use |
|---|---|
| Direct TCP, HTTP, or HTTPS access to a private host, no header injection | PrivateResource |
| Header injection (credentials, deny rules) on an external hostname | EgressRule with `domain_pattern` |
| Header injection on a private resource (e.g., per-agent token to an internal GitLab) | **Not supported in v1** — see [open-questions](../../open-questions.md) |

If a private resource needs per-agent credentials in v1, the operator passes them via the agent's ENVs (existing [Secrets](../../architecture/secrets.md) path) — the agent's tool adds the Authorization header itself.

## Observability

Tunnel liveness is surfaced per credential in the Console: `Online`, `Offline`, `Never enrolled`, with last-seen timestamps sourced from the OpenZiti Controller's session info. A Network's overall reachability is derived: a resource is **reachable** if at least one of its Network's tunnels is online, **degraded** otherwise.

Per-request tracing and metering for traffic through PrivateResources are not part of v1 — see [open-questions](../../open-questions.md).

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
| Tunnel credential revoked | The tunneler holding it loses its session immediately. Other tunnels in the Network are unaffected |
| Resource deleted | All access grants on the resource are cascaded-deleted. In-flight connections reset |
| Network deleted | All tunnel credentials, resources, and access grants in the Network are cascaded-deleted |
| Group membership changes | All resources granted to that group reflect the new membership within the propagation window |

## Constraints

- A resource has a single protocol (`tcp`, `http`, or `https`). UDP is not supported in v1. Mixed-protocol targets on the same host = multiple resources.
- A resource may declare multiple ports as a list (e.g., `[5432]`, `[80, 443]`, `[9200, 9300]`), but port ranges are not supported in v1. The intercept-to-target port mapping is positional 1:1 and the protocol is shared.
- Reserved zones are rejected on `intercept_host`: `*.ziti`, `*.svc`, `*.cluster.local`, any pattern overlapping `100.64.0.0/10`, `localhost`, `127.0.0.0/8`, and `::1/128`.
- For each port in `intercept_ports`, the tuple `(intercept_host, port)` must be unique across all resources in the organization. Operators namespace by hostname (`prod-postgres.corp:5432` vs `dev-postgres.corp:5432`).
- A Tunnel belongs to one Network. Running one tunneler for two networks requires two separate tunneler installations.
- Runners are not eligible group members and not eligible access principals — they are infrastructure, not actors in the access model.
- A private resource conflict with an EgressRule on the same hostname surfaces as an OpenZiti Controller error at the second `CreateService` call — no friendly cross-primitive detection in v1.

## Related Architecture

- [Private Networks](../../architecture/private-networks.md) — architecture overview
- [Networks Service](../../architecture/networks-service.md) — control-plane CRUD, reconciliation, OpenZiti resource lifecycle
- [Groups Service](../../architecture/groups-service.md) — Group / GroupMembership lifecycle
- [Resource Definitions](../../architecture/resource-definitions.md) — canonical schemas for Network, TunnelCredential, PrivateResource, PrivateResourceAccess, Group, GroupMembership
- [OpenZiti Integration](../../architecture/openziti.md) — overlay infrastructure, tunnel identity lifecycle, group role-attribute sync
- [Egress Gateway](../egress-gateway/egress-gateway.md) — sibling primitive for outbound HTTP/HTTPS to external destinations
- [Authorization](../../architecture/authz.md) — OpenFGA `group` type and per-service authorization checks
