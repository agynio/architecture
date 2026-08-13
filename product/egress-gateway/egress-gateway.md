# Egress Gateway

## Purpose

The Egress Gateway lets operators control and shape outbound HTTP/HTTPS traffic from agent workloads — adding credentials on the fly and selectively blocking destinations — without putting secrets into the agent container or modifying the agent's tools.

Two capabilities:

- **Credential injection** — a request from the agent to a specific destination (e.g., `api.github.com`) has a header (e.g., `Authorization: Bearer …`) added by the platform before it reaches the destination. The secret value never enters the agent container's environment.
- **Selective denial** — a request matching a configured pattern is rejected by the platform with `403`, regardless of what the agent's tooling tries to do.

Both work the same way whether the destination is on the public internet or inside the operator's own network. A rule names either a public hostname pattern or one of the organization's [private resources](../private-networks/private-networks.md), and everything else about the rule is identical.

The agent runs unmodified tools (`curl`, `git`, language-specific HTTP clients). The interception is transparent — the agent's code does not need to know the gateway exists.

## User Stories

- As an operator, I want my agent to make authenticated calls to GitHub without giving the agent the GitHub token, so a compromised agent process cannot exfiltrate the credential.
- As an operator, I want the same for my internal GitLab, without learning a second mechanism because the host happens to sit inside my VPC.
- As an operator, I want to attach the same credential-injection rule to several agents at once and rotate the underlying secret in one place.
- As an operator, I want to block agents from reaching specific destinations (e.g., a competitor's API, a known data-exfiltration endpoint).
- As an operator, I want an agent to read from an internal API but never `DELETE` against it.
- As an operator, I want to see, per agent, which destinations have been called and which were blocked.

## Concepts

| Term | Definition |
|---|---|
| **Egress Rule** | A rule that opts a destination into platform-mediated egress, optionally narrowed by method and path. A rule can permit or deny matching requests and can inject HTTP headers (literal values or [Secret](../../architecture/providers.md#secret) references). |
| **Destination** | What the rule applies to: a **public** hostname pattern (`api.github.com`, `*.github.com`), or a **private** [resource](../private-networks/private-networks.md) behind one of the organization's tunnels, chosen from a list. Every rule has exactly one, chosen at create time and fixed thereafter. |
| **Rule Attachment** | A relationship binding a rule to a specific agent or to an [environment](../environments/environments.md). One rule can be attached to many targets; one target can have many rules. Environment attachments apply to every workload running the environment — agent workloads and [sandboxes](../sandboxes/sandboxes.md) alike. |
| **Platform CA** | A platform-managed certificate authority whose certificate is installed in every agent container's trust store. Allows the gateway to terminate TLS for inspected requests without breaking certificate validation. |

## What gets intercepted

Only HTTP and HTTPS traffic to destinations matched by at least one attached rule's matcher is routed through the Egress Gateway. Traffic to public destinations with no matching rule flows directly from the agent container — the gateway does not see it and does not influence it.

This is **default-allow at the egress layer** for public destinations. Rules are the units that opt them into the gateway. A rule with `effect.action: deny` is the way to block a specific one; there is no global "block everything not listed" setting on this feature. (Operators that need a strict deny-all-then-allow posture can compose this feature with a [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) at the cluster layer; the egress feature itself stays default-allow.)

**Private destinations are default-deny and always were.** An agent reaches a private resource only if something says it may — an access grant on the resource, or a rule naming it attached to that agent. This is the one behavior that genuinely differs between the two destination types, and it is a property of private networking, not of rules.

### Scope of interception

Egress rules apply to outbound HTTP/HTTPS from **any container in the agent's pod** — the agent itself and its MCP sidecars. They share the pod's network namespace and therefore share the Ziti sidecar's interception. Operators writing rules for an agent should expect MCP outbound calls to be intercepted the same way the agent's are. There is no per-container scoping in v1.

### What is never intercepted

- `.agyn` hostnames (platform services: Gateway, LLM Proxy, Tracing, exposed services) — routed via their own OpenZiti services.
- Cluster-internal addresses — blocked by the workload-namespace NetworkPolicy installed with the runner (see "Workload network policy" below).
- Pod-local (`localhost`, MCP sidecars accessed via loopback) — never leaves the pod.

### Workload network policy

The runner's installation includes a NetworkPolicy in the workload namespace that restricts every agent workload pod's egress to:

- The OpenZiti synthetic range (`100.64.0.0/10`) — for platform services and matched egress rules.
- Cluster DNS — needed to resolve non-`.agyn` hostnames the agent legitimately calls (the resolved IP flows through the Ziti sidecar's interception for matched rules, or directly to public internet for unmatched destinations).
- Public internet — for unmatched destinations and for the Ziti sidecar to reach edge routers.

The policy **blocks** access to cluster pod CIDRs, cluster service CIDRs, and any operator-declared internal CIDRs. Agents cannot reach other in-cluster services from within the workload pod — the only platform-managed egress paths are `.agyn` services and rule-matched destinations.

The policy is static infrastructure parameterized at runner install time, not per workload — see [k8s-runner — Workload Egress NetworkPolicy](../../architecture/k8s-runner.md#workload-egress-networkpolicy).

## Rule shape

A rule has two parts: a **matcher** that says which requests it applies to, and an **effect** that says what happens to those requests. One rule per destination — `(organization, domain pattern)` for a public rule, `(organization, private resource)` for a private one. Uniqueness lets each rule map cleanly to one interception target.

### Matcher

The first choice when creating a rule is the destination type. It determines one field and nothing else.

| Field | Description |
|---|---|
| **Destination type** | `Public` or `Private`. Fixed at create time — changing where a rule points means deleting it and writing a new one. |
| **Domain pattern** | *Public only.* The hostname the rule applies to. Examples: `api.github.com`, `*.github.com`. Subdomain wildcards supported. Unique per organization. Reserved zones — `*.agyn`, `*.svc`, `*.cluster.local`, and the `100.64.0.0/10` synthetic range — are rejected, as is a hostname already claimed by a private resource. |
| **Ports** | *Public only.* List of destination ports to intercept (e.g., `[443]`, `[80, 443]`, `[8443]`). Defaults to `[80, 443]` when unset. |
| **Private resource** | *Private only.* One of the organization's [private resources](../private-networks/private-networks.md), picked from a list. Only `http` and `https` resources are offered — a `tcp` resource carries no HTTP requests to act on. The rule covers every port the resource declares. |
| **Methods** | List of HTTP methods the rule applies to (e.g., `["GET", "HEAD"]`). Empty means any method. |
| **Path pattern** | Glob over the request path (e.g., `/repos/**`, `/users/*/issues`). Empty means any path. |

### Upstream TLS (private destinations)

Public destinations present publicly-trusted certificates, and the platform verifies them normally. Internal ones often do not — a corporate CA, a self-signed certificate, or a certificate issued for the host's real name rather than the hostname agents dial. Three optional settings on a rule with an `https` private destination cover it:

| Field | Description |
|---|---|
| **Server name** | The hostname the platform presents and verifies against on its connection to the target. Defaults to the hostname agents dial. |
| **CA bundle** | A [Secret](../../architecture/providers.md#secret) holding the PEM bundle to verify the target against. |
| **Skip verification** | Accept any certificate the target presents. Mutually exclusive with a CA bundle. |

Left unset, the platform verifies exactly as it does for a public destination. A target with a private certificate then fails loudly rather than being trusted silently.

### Effect

| Field | Description |
|---|---|
| **Action** | `allow`, `deny`, or unset. Unset means the rule does not influence reachability (typical for injection-only rules); the request passes through to injection evaluation. |
| **Inject** | List of headers to inject on matching requests. Empty means no injection. See [Injected headers](#injected-headers). |

A rule whose effect has neither `action` nor `inject` set does nothing useful and surfaces as a validation warning at create time.

#### Injected headers

Each injected header has a name, an optional auth **scheme**, and a credential that is either a literal value or a [Secret](../../architecture/providers.md#secret) reference.

| Field | Description |
|---|---|
| **Name** | Header name (e.g., `Authorization`, `X-Api-Key`). |
| **Scheme** | `bearer`, `basic`, or unset. When set, the emitted header is `<Scheme> <credential>`. When unset, the credential is emitted as-is. |
| **Value / Secret** | The credential — a literal value or a reference to a Secret (resolved at request time). Exactly one. A referenced Secret must belong to the rule's organization. |

| Scheme | Credential | Emitted header |
|---|---|---|
| `bearer` | a GitHub token Secret | `Authorization: Bearer ghp_…` |
| `basic` | base64 of `user:pass` | `Authorization: Basic dXNlcjpwYXNz` |
| unset | an API key Secret | `X-Api-Key: …` |

For `basic`, the credential is the base64 encoding of `user:pass` — the platform does not encode it for you.

### Outcome matrix

| `effect.action` | `effect.inject` | What happens to a matching request |
|---|---|---|
| `allow` | empty | Permit; pass-through. |
| `allow` | non-empty | Permit and inject the headers. |
| `deny` | (any) | Reject with `403`; injection ignored. |
| unset | non-empty | Pass-through with injection. |
| unset | empty | No behavioral effect — flagged at create time. |

### Multiple rules in scope

When several rules attached to the same agent could match a request:

- **Reachability:** any matching rule with `effect.action: deny` wins (deny beats allow).
- **Injection:** headers from every matching rule's `effect.inject` are merged. On header-name collision, the rule with the lexicographically later `id` wins.

The original `Authorization` header (or any other header the agent's tool happens to set) is overwritten by an injection that targets the same header name — same precedence rule the [LLM Proxy](../../architecture/llm-proxy.md#header-forwarding) uses today.

## Attaching rules to agents

Rules are organization-scoped resources, independent of any agent. Attaching a rule to an agent enables it for that agent's workloads. A rule may instead be attached to an [environment](../environments/environments.md), enabling it for every workload running that environment — including [sandboxes](../sandboxes/sandboxes.md), which have no agent to attach rules to. The effective rules for a workload are the union of its agent's attachments (if any) and its environment's attachments. A rule with no attachments has no effect; a workload whose agent and environment have no attached rules makes no use of the egress gateway.

| Operation | Effect |
|---|---|
| Attach a rule to an agent | The rule begins applying to that agent's outbound requests within the propagation window (see below). For a private destination, the agent also gains access to the resource. |
| Detach a rule | The rule stops applying. For a private destination, the access it granted goes with it. Existing in-flight connections that depended on the rule are torn down; the agent's tool sees a connection reset. |
| Edit a rule | Same propagation as attachment — the next request the gateway sees uses the updated rule. |

**Propagation window: ≤15 seconds** from a rule change to the next request being affected. The 10-second OpenZiti tunneler poll interval is the dominant term.

### Attaching a private-destination rule grants access

For a public destination the agent could already reach the host; the rule only shapes what happens on the way. For a private one there is nothing to shape until the agent is allowed to connect at all, so **attaching the rule grants that access, and detaching it takes the access back**. One act, one place, and the credential arrives with the reachability rather than in a separate step.

This grants no more than the alternative already did: adding an agent to a resource's [access list](../private-networks/private-networks.md#granting-access) requires `can_edit_config` on that agent, and so does attaching a rule to it. The same is true of environments, with the same consequence — a grant to an environment reaches every sandbox anyone can start in it.

The resource's access list stays the answer to "who can reach this?", and it shows both sources: principals granted directly, and principals reaching it through an attached rule. Revoking is done wherever the access came from — remove the grant, or detach the rule.

Access lists remain the only way in for a user's device, an app, or a group, and the only way in at all for a `tcp` resource. Rules attach to agents and environments; those are the principals that run HTTP-speaking workloads.

## Behavior the agent observes

When a rule matches a request:

| Effect | What the agent's tool sees |
|---|---|
| `action: allow`, no inject | Normal response from the destination, transparent. |
| `action: allow`, with inject | Normal response from the destination — the platform injected the credentials before the request left. |
| `action: deny` | HTTP `403 Forbidden` from the platform, returned in place of the destination's response. The body identifies the deny as a platform action (no information about which rule, to avoid leaking policy details to the agent). |

When no rule covers the destination for this caller:

- **Public destination:** the connection bypasses the gateway entirely. The agent's tool reaches the destination directly. No headers are injected. No policy is applied.
- **Private destination the caller reaches through an access grant:** the connection passes through the platform untouched — not decrypted, not parsed, no certificate substituted. A rule written for one agent does not change what any other caller sees, which matters most for callers that never trusted the Platform CA in the first place, like a developer's laptop dialing over its own device identity.
- **Private destination the caller cannot reach at all:** the connection fails, as it does for any ungranted principal.

"Covers" is per destination, not per request. A caller with a rule attached for a destination has its connections there inspected even when the rule's methods or path pattern do not match the particular request — the platform must read the request to know that. Such a request is then passed through unchanged.

## TLS interception

To inspect HTTPS requests and inject credentials, the platform performs **TLS interception** on traffic routed through the gateway. The agent container's trust store is configured to trust the **Platform CA**; when the gateway terminates a TLS connection, it presents a certificate signed by that CA so the agent's client validates it normally.

Limitations:

- **Tools that hardcode TLS trust** (some mobile-style SDKs, some compiled tools with pinned certificates) cannot be intercepted. Their requests will fail TLS verification when routed through the gateway. Such tools must be updated to use the standard system trust store, or the destination they call must not have any matched rule.
- **Tools that use a non-standard system trust path** (Java's `cacerts`, .NET's per-runtime store) require the user's container image to install the Platform CA into the appropriate path. The platform sets the standard env vars (`SSL_CERT_FILE`, `REQUESTS_CA_BUNDLE`, `NODE_EXTRA_CA_CERTS`, `CURL_CA_BUNDLE`) — clients that read these work transparently.
- **Tools that bypass the platform DNS resolver** (configuring a custom resolver like `dig @8.8.8.8`, or a Go program with an explicit `net.Resolver`) skip the Ziti sidecar's DNS interception, get the real public IP, and connect to it directly. The egress rule does not apply. The agent's tool has not been credentialed by the platform, so no privilege escalation occurs — but the request is uninterceptable. Tools that use the system DNS resolver (the default for `curl`, `requests`, `httpx`, `node-fetch`, Go's default resolver) are unaffected.
- **WebSockets are not supported in v1.** Requests with `Upgrade: websocket` are refused by the gateway with `426 Upgrade Required`.
- **HTTP/3 / QUIC is not supported** — the entire interception model is TCP-based.

## Observability

Every outbound request handled by the gateway emits a tracing span ([Tracing](../../architecture/tracing.md)) and a metering record ([Metering](../../architecture/metering.md)).

| Span attribute | Description |
|---|---|
| `egress.method` | HTTP method |
| `egress.host` | Destination hostname (e.g., `api.github.com`, or the hostname agents dial for a private resource) |
| `egress.path` | Request path (query string stripped) |
| `egress.outcome` | `allow`, `deny`, `upstream_error` |
| `egress.private_resource_id` | The private resource, on private-destination traffic |
| `egress.matched_rule_ids` | The rules whose effect or injection were applied |
| `agyn.agent.id` | The agent the request originated from |

Header values, request bodies, and response bodies are **not** recorded. The span is sufficient to answer "which agent called which destination when, and was it allowed."

These spans are visible in the Console's tracing view, filterable by `agyn.agent.id`, `egress.host`, and `egress.outcome` — this is how an operator sees, per agent, which destinations were called and which were blocked. Private destinations covered by a rule appear here too; private traffic with no rule on it still does not, since nothing on the platform inspects it.

## Lifecycle

Rules are created, edited, and deleted by organization owners through the Console (see [Console — Egress Rules](../console/console.md#egress-rules)), the `agyn` CLI, or the Terraform provider. Attaching and detaching rules requires the same permission as editing the target's configuration — `can_edit_config` on the agent, or `can_edit_config` on the [environment](../environments/environments.md#who-can-use-an-environment).

| Event | Effect |
|---|---|
| Rule created | No effect on traffic until attached to an agent. For a private destination, the first rule on a resource moves that resource's traffic onto the gateway — see below. |
| Rule attached to agent | Applies on the next workload start; existing workloads pick it up within the propagation window. For a private destination, also grants access. |
| Rule edited | Same propagation as attachment. |
| Rule detached | Stops applying within the propagation window. For a private destination, also revokes the access it granted. |
| Rule deleted | Fails if the rule is attached to one or more agents or environments — detach first. Deleting the last rule on a private resource moves that resource's traffic back off the gateway. |
| Private resource deleted | Fails while any rule names it — delete those rules first. |
| Secret referenced by a rule rotates | The new value is used on the next request after rotation. |

**Creating the first rule on a private resource, or deleting its last one, briefly interrupts connections to that resource — for every caller, not only the ones the rule is attached to.** The platform is moving that resource's traffic onto or off the gateway; live connections reset and tools reconnect. It is the same interruption a detach already causes, within the same ≤15s window, and the Console says so before either action.

## Constraints

- Only HTTP and HTTPS are subject to interception. Other protocols (raw TCP, SMTP, database wire protocols) are out of scope for v1. A `tcp` private resource cannot be named by a rule; reach it through an access grant and pass credentials another way.
- The agent's container image must trust the Platform CA via one of the standard mechanisms (env-var-honoring HTTP client, or CA installed into the system trust store).
- Wildcard patterns in `matcher.domain_pattern` cover one subdomain segment (`*.github.com` matches `api.github.com` but not `code.api.github.com`); multi-segment wildcards are out of scope for v1.
- Two rules cannot share a destination — the same `(organization, domain pattern)`, or the same private resource. Express finer-grained method or path policies within a single rule's `matcher.methods` and `matcher.path_pattern`. (Future: per-condition sub-policies inside one rule.) One consequence for private destinations: every agent attached to a resource's rule gets the same injected credential, so per-agent credentials to one internal host are not expressible in v1.
- A rule's destination type is fixed at create time. Repointing a rule from public to private, or between resources, means deleting it and creating another.
- Reserved domain patterns are rejected at create time: `*.agyn`, `*.svc`, `*.cluster.local`, and any pattern overlapping the OpenZiti synthetic range (`100.64.0.0/10`). A domain pattern equal to a private resource's hostname is rejected too, pointing at a private-destination rule instead.
- Cluster-internal services are not reachable from agent workloads regardless of rule configuration — the platform's NetworkPolicy blocks them at the cluster network layer.
- Secrets referenced by `effect.inject` headers are not auto-refreshed. Rotating the secret value via the [Secrets](../../architecture/secrets.md) service takes effect on the next request. Tokens that require an active refresh (OAuth access tokens, short-lived STS credentials) must be refreshed externally.

## Related Architecture

- [Egress Gateway](../../architecture/egress-gateway.md)
- [Private Networks — Gateway Mediation](../../architecture/private-networks.md#gateway-mediation)
- [Product — Private Networks](../private-networks/private-networks.md)
- [Secrets](../../architecture/secrets.md)
- [Resource Definitions — Egress Rule](../../architecture/resource-definitions.md#egress-rule)
- [OpenZiti Integration](../../architecture/openziti.md)
- [Authorization](../../architecture/authz.md)
