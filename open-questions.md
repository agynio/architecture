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
- What are the complete relation definitions for all resource types (models, workspaces, MCP servers, secrets)? How do org-scoped vs independent resources differ in their relation definitions?
- How are tracing data permissions modeled? (Organization-level visibility, or per-agent restriction?)
- How are Notifications subscriptions authorized? (Implicitly via participant membership, or explicit check?)
- What is the relationship tuple cleanup strategy when resources are deleted?

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

## Tracing AuthN/AuthZ

**Context:** The Tracing service exposes two gRPC interfaces: the standard OTLP `TraceService/Export` for ingestion and the custom `TracingService` for queries. Ingestion is accessed directly by producers (agents) without going through the Gateway. The query API is proxied through the Gateway via `TracingGateway`. Neither interface currently has authentication or authorization.

**Questions:**
- How is the ingestion endpoint authenticated? (Istio mTLS for internal producers? Service tokens for external producers? Unauthenticated within the cluster?)
- How is the ingestion endpoint authorized? (Any authenticated producer can export spans? Scoped to specific agents or organizations?)
- How are query results scoped? (Organization-level visibility, per-agent restriction, or full access for any authenticated user?)

---

## Codex LLM Endpoint Configuration: `OPENAI_BASE_URL` vs Custom Provider

**Context:** `agynd` currently configures Codex to use the [LLM Proxy](architecture/llm-proxy.md) by writing a custom model provider in `$CODEX_HOME/config.toml` (see [LLM Proxy — Agent Configuration](architecture/llm-proxy.md#agent-configuration)). The alternative is setting the `OPENAI_BASE_URL` environment variable to override the built-in OpenAI provider.

The custom provider approach was chosen because the built-in OpenAI provider triggers behaviors the LLM Proxy does not implement (remote compaction via `POST /responses/compact`, realtime WebSocket) and has `env_key: None` which prevents `OPENAI_API_KEY` from being used for Bearer authentication. However, this reasoning is based on the current Codex CLI behavior and may change as Codex evolves.

**Questions:**
- Does `OPENAI_BASE_URL` work correctly with `codex app-server`, or only with the interactive CLI?
- If Codex adds proper `OPENAI_BASE_URL` support for `app-server` (respecting `env_key` and disabling provider-specific behaviors), should we switch to it for simplicity?
- Are there other Codex provider-specific behaviors beyond compaction and WebSocket that the custom provider avoids?
