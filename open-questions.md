# Open Questions

Unresolved architectural decisions requiring discussion.

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

**Context:** Future capability. Agents should be able to expose a port and share it with specific other agents over a private OpenZiti connection. Other agents must not be able to connect. See [OpenZiti Integration — Dynamic Policies](architecture/openziti.md#dynamic-policies-future).

**Questions:**
- How does the agent request port sharing? (Platform API tool? Agent SDK primitive?)
- What is the resource model? (Does the platform define "agent networks" as a team resource, or is each share ad-hoc?)
- Does the binding agent create the OpenZiti service dynamically, or does the Ziti Management service pre-provision it?

---

## Authorization Model Completeness

**Context:** The authorization approach is decided — OpenFGA with a dedicated Authorization service (see [Authorization](architecture/authz.md)). The initial model covers threads, files, and organization roles. The full model needs to cover all resource types and access patterns.

**Questions:**
- What are the complete relation definitions for all resource types (models, workspaces, MCP servers, channels, secrets)? How do org-scoped vs independent resources differ in their relation definitions?
- How are tracing data permissions modeled? (Organization-level visibility, or per-agent restriction?)
- How are Notifications subscriptions authorized? (Implicitly via participant membership, or explicit check?)
- What is the relationship tuple cleanup strategy when resources are deleted?

---

## agn Configuration Structure

**Context:** [`agn`](architecture/agn-cli.md) configuration lives in `~/.agyn/agn/config.yaml`. The minimal configuration (LLM endpoint, system prompt) is defined — see [agn Configuration](architecture/agn-cli.md#configuration). Remaining questions are about features not yet implemented.

**Questions:**
- What is the directory convention for skills on the filesystem?
- How does `agn` discover its MCP server? (Stdio command, socket path, environment variable?)
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

**Context:** [Apps](architecture/apps.md) currently use cluster-level permissions — a cluster-scoped app can send messages to any thread in the platform. This is acceptable for self-hosted deployments where the platform operator controls which apps are installed.

**Questions:**
- When should permissions narrow to org-level? (When org-scoped apps are introduced?)
- Should thread-level permissions exist? (App X can only write to threads where an agent explicitly granted access?)
- How does permission delegation work if an agent grants an app access to a specific thread?
- What is the authorization check path for app → thread operations? (Direct OpenFGA check, or mediated by a service?)

---

## Org-Scoped Apps

**Context:** The current design covers cluster-scoped apps deployed as platform infrastructure. Future needs include org-scoped apps managed by organization administrators. (Runner org-scoping is handled by the [Runners](architecture/runners.md) service.)

**Questions:**
- How does an org admin register an org-scoped app? (Via `agyn` CLI with org context? Via Terraform?)
- How are org-scoped and cluster-scoped apps with the same slug resolved? (Org-scoped takes precedence? Cluster slugs are reserved?)
- How does the enrollment flow differ for org-scoped apps? (Same service token flow, scoped to the org?)
- Are org-scoped apps visible only within their organization, or can they be shared?

---

## Runner Selection Strategy

**Context:** With multiple runners (cluster-scoped and org-scoped) registered in the [Runners](architecture/runners.md) service, the [Agents Orchestrator](architecture/agents-orchestrator.md) needs to choose which runner handles a given agent workload.

**Questions:**
- What criteria determine runner selection? (Organization affinity? Agent resource definition? Labels/tags? Capacity?)
- Can an agent resource definition specify a runner preference or requirement?
- What is the fallback behavior if a preferred runner is unavailable?

---

## Per-Runner OpenZiti Addressing

**Context:** The [Runners](architecture/runners.md) service tracks which runner hosts each workload. The [Terminal Proxy](architecture/terminal-proxy.md) and [Agents Orchestrator](architecture/agents-orchestrator.md) need to reach a specific runner instance. Currently, all runners bind the same `runner` OpenZiti service — `Dial("runner")` load-balances across instances with no mechanism to target a specific one.

**Questions:**
- How does a caller dial a specific runner? Options: (a) per-runner OpenZiti services (e.g., `runner-{runnerId}`) created dynamically at registration, (b) OpenZiti `appdata` or terminator-level identity matching, (c) a different addressing mechanism.
- If per-runner services, who creates them? The [Runners](architecture/runners.md) service during registration via [Ziti Management](architecture/openziti.md)?
- How do static Dial policies work with dynamic per-runner services?

---

## Tracing AuthN/AuthZ

**Context:** The Tracing service exposes two gRPC interfaces: the standard OTLP `TraceService/Export` for ingestion and the custom `TracingService` for queries. Ingestion is accessed directly by producers (agents) without going through the Gateway. The query API is proxied through the Gateway via `TracingGateway`. Neither interface currently has authentication or authorization.

**Questions:**
- How is the ingestion endpoint authenticated? (Istio mTLS for internal producers? Service tokens for external producers? Unauthenticated within the cluster?)
- How is the ingestion endpoint authorized? (Any authenticated producer can export spans? Scoped to specific agents or organizations?)
- How are query results scoped? (Organization-level visibility, per-agent restriction, or full access for any authenticated user?)
