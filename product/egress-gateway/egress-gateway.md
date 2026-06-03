# Egress Gateway

## Purpose

The Egress Gateway lets operators control and shape outbound HTTP/HTTPS traffic from agent workloads — adding credentials on the fly and selectively blocking destinations — without putting secrets into the agent container or modifying the agent's tools.

Two capabilities:

- **Credential injection** — a request from the agent to a specific destination (e.g., `api.github.com`) has a header (e.g., `Authorization: Bearer …`) added by the platform before it reaches the destination. The secret value never enters the agent container's environment.
- **Selective denial** — a request matching a configured pattern is rejected by the platform with `403`, regardless of what the agent's tooling tries to do.

The agent runs unmodified tools (`curl`, `git`, language-specific HTTP clients). The interception is transparent — the agent's code does not need to know the gateway exists.

## User Stories

- As an operator, I want my agent to make authenticated calls to GitHub without giving the agent the GitHub token, so a compromised agent process cannot exfiltrate the credential.
- As an operator, I want to attach the same credential-injection rule to several agents at once and rotate the underlying secret in one place.
- As an operator, I want to block agents from reaching specific destinations (e.g., a competitor's API, a known data-exfiltration endpoint).
- As an operator, I want to see, per agent, which destinations have been called and which were blocked.

## Concepts

| Term | Definition |
|---|---|
| **Egress Rule** | A rule that opts a destination (matched by domain pattern, optionally narrowed by method and path) into platform-mediated egress. A rule can permit or deny matching requests and can inject HTTP headers (literal values or [Secret](../../architecture/providers.md#secret) references). |
| **Rule Attachment** | A relationship binding a rule to a specific agent. One rule can be attached to many agents; one agent can have many rules. |
| **Platform CA** | A platform-managed certificate authority whose certificate is installed in every agent container's trust store. Allows the gateway to terminate TLS for inspected requests without breaking certificate validation. |

## What gets intercepted

Only HTTP and HTTPS traffic to destinations matched by at least one attached rule's matcher is routed through the Egress Gateway. Traffic to destinations with no matching rule flows directly from the agent container — the gateway does not see it and does not influence it.

This is **default-allow at the egress layer.** Rules are the units that opt destinations into the gateway. A rule with `effect.action: deny` is the way to block a specific destination; there is no global "block everything not listed" setting on this feature. (Operators that need a strict deny-all-then-allow posture can compose this feature with a [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) at the cluster layer; the egress feature itself stays default-allow.)

### Scope of interception

Egress rules apply to outbound HTTP/HTTPS from **any container in the agent's pod** — the agent itself, MCP sidecars, and hooks. They share the pod's network namespace and therefore share the Ziti sidecar's interception. Operators writing rules for an agent should expect MCP and hook outbound calls to be intercepted the same way the agent's are. There is no per-container scoping in v1.

### What is never intercepted

- `.ziti` hostnames (platform services: Gateway, LLM Proxy, Tracing, exposed services) — routed via their own OpenZiti services.
- Cluster-internal addresses — blocked by the workload-namespace NetworkPolicy installed with the runner (see "Workload network policy" below).
- Pod-local (`localhost`, MCP sidecars accessed via loopback) — never leaves the pod.

### Workload network policy

The runner's installation includes a NetworkPolicy in the workload namespace that restricts every agent workload pod's egress to:

- The OpenZiti synthetic range (`100.64.0.0/10`) — for platform services and matched egress rules.
- Cluster DNS — needed to resolve non-`.ziti` hostnames the agent legitimately calls (the resolved IP flows through the Ziti sidecar's interception for matched rules, or directly to public internet for unmatched destinations).
- Public internet — for unmatched destinations and for the Ziti sidecar to reach edge routers.

The policy **blocks** access to cluster pod CIDRs, cluster service CIDRs, and any operator-declared internal CIDRs. Agents cannot reach other in-cluster services from within the workload pod — the only platform-managed egress paths are `.ziti` services and rule-matched destinations.

The policy is static infrastructure parameterized at runner install time, not per workload — see [k8s-runner — Workload Egress NetworkPolicy](../../architecture/k8s-runner.md#workload-egress-networkpolicy).

## Rule shape

A rule has two parts: a **matcher** that says which requests it applies to, and an **effect** that says what happens to those requests. One rule per `(organization, matcher.domain_pattern)` — uniqueness lets each rule map cleanly to one OpenZiti interception target.

### Matcher

| Field | Description |
|---|---|
| **Domain pattern** | The hostname the rule applies to. Examples: `api.github.com`, `*.github.com`. Subdomain wildcards supported. Required. Unique per organization. Reserved zones — `*.ziti`, `*.svc`, `*.cluster.local`, and the `100.64.0.0/10` synthetic range — are rejected. |
| **Ports** | List of destination ports to intercept (e.g., `[443]`, `[80, 443]`, `[8443]`). Defaults to `[80, 443]` when unset. |
| **Methods** | List of HTTP methods the rule applies to (e.g., `["GET", "HEAD"]`). Empty means any method. |
| **Path pattern** | Glob over the request path (e.g., `/repos/**`, `/users/*/issues`). Empty means any path. |

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

Rules are organization-scoped resources, independent of any agent. Attaching a rule to an agent enables it for that agent's workloads. A rule with no attachments has no effect; an agent with no attached rules makes no use of the egress gateway.

| Operation | Effect |
|---|---|
| Attach a rule to an agent | The rule begins applying to that agent's outbound requests within the propagation window (see below). |
| Detach a rule | The rule stops applying. Existing in-flight connections that depended on the rule are torn down; the agent's tool sees a connection reset. |
| Edit a rule | Same propagation as attachment — the next request the gateway sees uses the updated rule. |

**Propagation window: ≤15 seconds** from a rule change to the next request being affected. The 10-second OpenZiti tunneler poll interval is the dominant term.

## Behavior the agent observes

When a rule matches a request:

| Effect | What the agent's tool sees |
|---|---|
| `action: allow`, no inject | Normal response from the destination, transparent. |
| `action: allow`, with inject | Normal response from the destination — the platform injected the credentials before the request left. |
| `action: deny` | HTTP `403 Forbidden` from the platform, returned in place of the destination's response. The body identifies the deny as a platform action (no information about which rule, to avoid leaking policy details to the agent). |

When no rule matches:

- The connection bypasses the gateway entirely. The agent's tool reaches the destination directly. No headers are injected. No policy is applied.

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
| `egress.host` | Destination hostname (e.g., `api.github.com`) |
| `egress.path` | Request path (query string stripped) |
| `egress.outcome` | `allow`, `deny`, `upstream_error` |
| `egress.matched_rule_ids` | The rules whose effect or injection were applied |
| `agyn.agent.id` | The agent the request originated from |

Header values, request bodies, and response bodies are **not** recorded. The span is sufficient to answer "which agent called which destination when, and was it allowed."

These spans are visible in the Console's tracing view, filterable by `agyn.agent.id`, `egress.host`, and `egress.outcome` — this is how an operator sees, per agent, which destinations were called and which were blocked.

## Lifecycle

Rules are created, edited, and deleted by organization owners through the Console (see [Console — Egress Rules](../console/console.md#egress-rules)), the `agyn` CLI, or the Terraform provider. Attaching and detaching rules to agents requires the same permission as editing the agent's configuration (`can_edit_config` on the agent).

| Event | Effect |
|---|---|
| Rule created | No effect on traffic until attached to an agent. |
| Rule attached to agent | Applies on the next workload start; existing workloads pick it up within the propagation window. |
| Rule edited | Same propagation as attachment. |
| Rule detached | Stops applying within the propagation window. |
| Rule deleted | Fails if the rule is attached to one or more agents — detach first. |
| Secret referenced by a rule rotates | The new value is used on the next request after rotation. |

## Constraints

- Only HTTP and HTTPS are subject to interception. Other protocols (raw TCP, SMTP, database wire protocols) are out of scope for v1.
- The agent's container image must trust the Platform CA via one of the standard mechanisms (env-var-honoring HTTP client, or CA installed into the system trust store).
- Wildcard patterns in `matcher.domain_pattern` cover one subdomain segment (`*.github.com` matches `api.github.com` but not `code.api.github.com`); multi-segment wildcards are out of scope for v1.
- Two rules cannot share the same `(organization, matcher.domain_pattern)`. Express finer-grained method or path policies within a single rule's `matcher.methods` and `matcher.path_pattern`. (Future: per-condition sub-policies inside one rule.)
- Reserved domain patterns are rejected at create time: `*.ziti`, `*.svc`, `*.cluster.local`, and any pattern overlapping the OpenZiti synthetic range (`100.64.0.0/10`).
- Cluster-internal services are not reachable from agent workloads regardless of rule configuration — the platform's NetworkPolicy blocks them at the cluster network layer.
- Secrets referenced by `effect.inject` headers are not auto-refreshed. Rotating the secret value via the [Secrets](../../architecture/secrets.md) service takes effect on the next request. Tokens that require an active refresh (OAuth access tokens, short-lived STS credentials) must be refreshed externally.

## Related Architecture

- [Egress Gateway](../../architecture/egress-gateway.md)
- [Secrets](../../architecture/secrets.md)
- [Resource Definitions — Egress Rule](../../architecture/resource-definitions.md#egress-rule)
- [OpenZiti Integration](../../architecture/openziti.md)
- [Authorization](../../architecture/authz.md)
