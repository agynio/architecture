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
- How are tracing data permissions modeled beyond the initial design? See [Tracing — Query Authorization](architecture/tracing.md#query-authorization) for the current approach.
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

**Context:** [Apps](architecture/apps.md) use [installation-based permissions](architecture/apps.md#permissions-bridge) — an installation grants the app broad permissions within the installing organization (create threads, add participants). Thread-level access comes from participant membership. Write-only apps receive `thread:write` within the org.

**Questions:**
- Should permissions narrow to per-agent or per-thread granularity? (e.g., installation config specifies agent X, but authorization allows interaction with any agent in the org)
- How does permission delegation work if an agent grants an app access to a specific thread?
- What is the authorization check path for app → thread operations? (Direct OpenFGA check, or mediated by a service?)

---

## Codex LLM Endpoint Configuration: `OPENAI_BASE_URL` vs Custom Provider

**Context:** `agynd` currently configures Codex to use the [LLM Proxy](architecture/llm-proxy.md) by writing a custom model provider in `$CODEX_HOME/config.toml` (see [LLM Proxy — Agent Configuration](architecture/llm-proxy.md#agent-configuration)). The alternative is setting the `OPENAI_BASE_URL` environment variable to override the built-in OpenAI provider.

The custom provider approach was chosen because the built-in OpenAI provider triggers behaviors the LLM Proxy does not implement (remote compaction via `POST /responses/compact`, realtime WebSocket) and has `env_key: None` which prevents `OPENAI_API_KEY` from being used for Bearer authentication. However, this reasoning is based on the current Codex CLI behavior and may change as Codex evolves.

**Questions:**
- Does `OPENAI_BASE_URL` work correctly with `codex app-server`, or only with the interactive CLI?
- If Codex adds proper `OPENAI_BASE_URL` support for `app-server` (respecting `env_key` and disabling provider-specific behaviors), should we switch to it for simplicity?
- Are there other Codex provider-specific behaviors beyond compaction and WebSocket that the custom provider avoids?

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
