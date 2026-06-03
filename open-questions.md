# Open Questions

Unresolved product and architectural decisions requiring discussion.

---

## Agent Batching Protocol

**Context:** Simple case is one container per agent invocation. For specific agents, a single instance processing multiple threads may be more efficient.

**Questions:**
- What triggers batch assignment? (Agent type config? Runtime heuristic?)
- How does a batched agent receive and route messages for different threads?
- How is isolation maintained between threads in a batch?
- What happens when one thread in a batch fails?

---

## OpenZiti: Agent-to-Agent Private Networking

**Context:** Future capability. Agents should be able to expose a port and share it with specific other agents over a private OpenZiti connection. Other agents must not be able to connect. See [OpenZiti Integration — Dynamic Policies](architecture/openziti.md#dynamic-policies). User-to-agent port exposure has been designed (see [Expose Service](architecture/expose-service.md)); this question is about the agent-to-agent variant.

**Questions:**
- How does the agent request port sharing? (Platform API tool? Agent SDK primitive?)
- What is the resource model? (Does the platform define "agent networks" as a team resource, or is each share ad-hoc?)
- Does the binding agent create the OpenZiti service dynamically, or does the Ziti Management service pre-provision it?

---


## agn Configuration Structure

**Context:** [`agn`](architecture/agn-cli.md) configuration lives in `~/.agyn/agn/config.yaml`. The minimal configuration (LLM endpoint, system prompt) is defined — see [agn Configuration](architecture/agn-cli.md#configuration). Remaining questions are about features not yet implemented.

**Questions:**
- What is the directory convention for skills on the filesystem?
- What is the schema for state persistence backend selection? (Flag, config field, environment variable?)

---

## OpenZiti: Service Identity Self-Enrollment Authorization

**Context:** Infrastructure services (Orchestrator, Runner, Gateway) self-enroll their OpenZiti identities by calling `ZitiManagement.RequestServiceIdentity()` over Istio. Internal services are authenticated by Istio mTLS (ServiceAccount identity). However, external runners also communicate with platform services — and they must not be able to request arbitrary service identities or agent identities for agents they do not manage. See [OpenZiti Integration — Service Identity Self-Enrollment](architecture/openziti.md#service-identity-self-enrollment).

**Problem:** The current design does not specify how Ziti Management authorizes identity requests. Without explicit controls:
- An external runner could call `RequestServiceIdentity` with `roleAttributes: ["orchestrators"]` or `["gateway-hosts"]` and obtain an identity with elevated access.
- An external runner could call `CreateAgentIdentity` for agents it does not manage.

**Questions:**
- How does Ziti Management authorize `RequestServiceIdentity` calls? Should it restrict which role attributes each caller is allowed to request (e.g., only runners can request `["runners"]`)?
- How does Ziti Management authorize `CreateAgentIdentity` calls from the Orchestrator vs. reject them from other callers?
- Is Istio `AuthorizationPolicy` (ServiceAccount-level) sufficient for internal callers, or does Ziti Management need application-level authorization logic?
- For external runners (which connect via OpenZiti, not Istio), what is the trust boundary? Should external runners be able to call Ziti Management at all, or should all identity management be mediated by an internal service?

---

## App Permission Model

**Context:** [Apps](architecture/apps.md) declare the [permissions they require](architecture/apps.md#permissions) from an extensible vocabulary (`thread:create`, `thread:write`, `participant:add`). The [installation](architecture/apps.md#permissions-bridge) grants the declared permissions within the installing organization by writing authorization tuples.

**Questions:**
- Should permissions support per-agent or per-thread granularity? (e.g., `participant:add` narrowed to specific agents rather than any agent in the org)
- How does permission delegation work if an agent grants an app access to a specific thread?
- What is the authorization check path for app → thread operations? (Direct OpenFGA check, or mediated by a service?)
- Should the permission vocabulary be formalized in a registry, or are they free-form strings validated at the Apps Service layer?

---

## Installation Configuration Secrets

**Context:** [App installations](architecture/apps.md#configuration) store configuration as a JSON object. Some configuration values are sensitive (e.g., Telegram bot tokens, API keys for external services). Currently, configuration values are stored as plain text.

**Questions:**
- Should configuration values reference the [Secrets](architecture/secrets.md) service (similar to how agent ENVs can reference secrets)?
- Should the entire configuration object be encrypted at rest, or only specific keys marked as sensitive?
- Should the app declare which configuration keys are sensitive (as part of a future configuration schema)?
- What is the access control model for configuration retrieval? (Only the app itself, or also org admins?)

---

## Console Monitoring: Storage and Usage Metrics

**Context:** The Console monitoring dashboard shows active workloads via `Runners.ListWorkloads(organization_id)` — resolved. The remaining Console monitoring features (storage view, token consumption, compute hours) are not yet designed.

**Resolved:**
- Active workloads — `ListWorkloads` on the [Runners](architecture/runners.md) service, exposed via Gateway. The workload record already contains organization, agent, runner, thread, status, containers, and timestamps.

**Questions:**
- How should persistent volume (PVC) state be tracked and exposed? The [Runner](architecture/runner.md) creates PVCs, but the [Runners](architecture/runners.md) service does not track them. Without tracking, GC/TTL of orphaned PVCs is not possible. This needs a design decision: should the Runners service track PVCs as a first-class resource, or is there a simpler approach?
- How should historical usage metrics (token consumption, compute hours) be aggregated and served? A dedicated metering/analytics service, per-service summary tables, or deferred until billing is prioritized?

---

---

## Port Exposure: Scoped Access Control

**Context:** The current [Expose Service](architecture/expose-service.md) design grants all identities on the OpenZiti network access to all exposed services via `#all` on Dial policies. This is intentionally broad for the initial implementation.

**Questions:**
- Should exposed ports be scoped to specific users? (e.g., only the user who started the conversation can access the exposed port)
- If scoped, should per-user role attributes (`user-<userId>`) be assigned to device identities, and per-exposure Dial policies target specific users?
- How does multi-user thread access (shared threads, teams) interact with exposure scoping?

---

## Port Exposure: TLS for Exposed Services

**Context:** Exposed services are accessed over `http://exposed-<id>.ziti:<port>`. The OpenZiti overlay provides encryption in transit (mTLS between Ziti endpoints), but the user's browser connects to the local Ziti tunnel via plain HTTP on `localhost`.

**Questions:**
- Is the lack of browser-visible HTTPS acceptable for the initial version?
- Should the platform provide an option for agents to serve TLS-terminated services (agent provides its own cert)?
- Would a local HTTPS proxy in the Ziti tunnel client be feasible?


---

## Egress Gateway: Strict Deny-All Mode

**Context:** The [Egress Gateway](architecture/egress-gateway.md) is default-allow at the egress layer — only destinations matched by an attached [EgressRule](architecture/resource-definitions.md#egress-rule) are routed through the gateway. Unmatched destinations flow direct via the agent pod's `eth0`. For operators who need a stricter posture ("agent can ONLY reach destinations explicitly allowed"), no opt-in mechanism is specified.

**Questions:**
- Should strict mode be expressed as a per-agent setting (`strict_egress: true`), per-organization, or cluster-wide?
- Implementation: a NetworkPolicy that blocks all egress except to the OpenZiti synthetic range and cluster DNS? Or a wildcard `egress` Ziti service that captures everything and lets the gateway enforce default-deny?
- The wildcard-Ziti approach surfaced operational issues during the v1 spike (controller-IP exclusions, transient routing failures); is the NetworkPolicy-only approach sufficient?

---

## Egress Gateway: Multi-Segment Wildcard Hostnames

**Context:** [EgressRule](architecture/resource-definitions.md#egress-rule) `matcher.domain_pattern` supports single-segment wildcards (`*.github.com`). Multi-segment wildcards (`**.github.com`, matching arbitrary depth) are not supported in v1 because `openziti/ziti-edge-tunnel` doesn't expose them through `intercept.v1`.

**Questions:**
- Is multi-segment matching a common-enough need to justify the work?
- If yes: do we add it via a separate match field (`matcher.domain_recursive: true`) or upstream a feature request to OpenZiti?

---

## Egress Gateway: Conditions List per Rule

**Context:** v1 enforces one `EgressRule` per `(organization, matcher.domain_pattern)`. The rule has a single matcher and a single effect. Operators who need "allow GET on *.github.com, deny POST/DELETE on *.github.com" cannot express it as two rules (uniqueness conflict) and cannot express it within one rule (single effect).

**Questions:**
- Extend the rule schema with a `conditions: [{ matcher_clause, effect_clause }]` list, where the top-level `matcher` provides the OpenZiti intercept boundary (domain + ports) and each condition narrows by method/path with its own effect?
- Or relax the uniqueness constraint and deduplicate at the OpenZiti-service layer (one Ziti service per unique domain, multiple rules share the service)?

---

## Egress Gateway: OAuth-Style Token Refresh for Injected Secrets

**Context:** [EgressRule.effect.inject](architecture/resource-definitions.md#egress-rule) headers can reference a [Secret](architecture/providers.md#secret) by `secret_id`. The gateway resolves the secret value at request time. If the secret is an OAuth access token that has expired, the request fails with `401` from the upstream — the platform does not refresh the token automatically.

**Questions:**
- Should the platform support refresh-token rotation for secrets used in injection rules?
- If yes, where does the refresh logic live — extension to the Secrets service, a new rotation worker, or per-rule refresh policy?
- What identifies a refreshable secret vs. a static one in the [Secret](architecture/providers.md#secret) resource?

---

## Egress Gateway: WebSocket Support

**Context:** The Egress Gateway refuses HTTP `Upgrade` requests (including WebSocket) with `426 Upgrade Required` in v1. After the `Upgrade` handshake, the connection becomes a bidirectional binary stream — header injection is irrelevant and per-message access checks don't fit the rule model.

**Questions:**
- Are WebSocket destinations a meaningful use case for the egress gateway?
- If yes: passthrough after the handshake (no per-message rule application; injection happens only on the upgrade request)? Or extend the rule model to apply at the WebSocket message layer?

---

## Egress Gateway: OpenZiti Control-Plane Scale

**Context:** The [EgressRules service](architecture/egress-rules-service.md) provisions one OpenZiti service per rule and one Dial policy per attachment. At scale (hundreds of rules per organization across many organizations, with multiple Egress Gateway replicas each binding all `egress-services`-tagged services), the OpenZiti Controller's service and policy counts grow accordingly.

**Questions:**
- What is the practical upper limit on OpenZiti services and policies in a single Controller deployment?
- Does the multi-replica gateway pattern (each replica binding every per-rule service) scale linearly, or does the Controller see contention?
- Is a per-org or per-region Controller sharding pattern needed, and if so when?

---

## Egress Gateway: Migration from ENV-Injected Secrets

**Context:** Existing agents that authenticate to third-party APIs today do so via secrets injected as ENV vars into the agent container. The Egress Gateway provides a more secure alternative (the agent process never holds the credential). No automated migration is specified.

**Questions:**
- Should the platform offer tooling to convert an ENV-based secret usage to an EgressRule with `effect.inject`?
- Is there a transitional period where both work, and how does the platform detect the duplication?
