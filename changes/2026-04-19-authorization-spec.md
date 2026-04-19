# Authorization Spec

## Target

- [Authorization](../architecture/authz.md)
- [LLM Proxy — Authorization](../architecture/llm-proxy.md#authorization)
- [LLM — Authorization](../architecture/llm.md#authorization)
- [Agents Orchestrator — Workload Assembly](../architecture/agents-orchestrator.md#workload-assembly)
- [Agents Service — Authorization](../architecture/agents-service.md#authorization)
- [Agents Service — agynd Startup Fetch](../architecture/agents-service.md#agynd-startup-fetch)
- [agynd — Environment Preparation](../architecture/agynd-cli.md#3-environment-preparation)
- [Agent Init — Changes Required](../architecture/agent-init.md#changes-required)
- [Secrets — Authorization](../architecture/secrets.md#authorization)
- [Apps Service — Authorization](../architecture/apps-service.md#authorization)
- [Chat — Authorization](../architecture/chat.md#authorization)
- [Expose Service — Authorization](../architecture/expose-service.md#authorization)
- [Notifications — Authorization](../architecture/notifications.md#authorization)
- [Runners — Authorization](../architecture/runners.md#authorization)
- [Threads — Authorization](../architecture/threads.md#authorization)

## Delta

### Authorization model

- `authz.md` expanded into a full specification: OpenFGA DSL for all types (`cluster`, `organization`, `thread`, `model`), relation semantics, computed relations, and cross-type derivations.
- Added `model` type with `can_use` (computed from org `member`) and `can_manage` (computed from org `owner`) relations.
- Added app installation permission tuples (`thread_create`, `thread_write`, `participant_add`) and how they flow into thread computed relations.
- Added Tuple Lifecycle table: which service writes/deletes each tuple and on which event.
- Added per-service authorization reference sections for all services.
- Removed "Authorization Model Completeness" open question — resolved.

### Explicit model authorization

- LLM Proxy: replaced indirect `Check(identity:<id>, member, organization:<model.org_id>)` with explicit `Check(identity:<id>, can_use, model:<model_id>)`.
- LLM Service: on `CreateModel`, writes `organization:<org_id>, org, model:<model_id>` to the Authorization service; on `DeleteModel`, deletes the same tuple.

### ENV var and secret isolation

- Orchestrator injects ALL environment variables (plain-text and resolved secret values) per-container at workload assembly time. Per-container isolation: agent ENVs go only into the agent container, MCP ENVs only into their respective sidecar.
- `ZITI_ENROLLMENT_JWT` injected into the Ziti sidecar container, not the agent container.
- `ListENVs` never returns resolved secret values — secret-backed ENVs return `{name, secret_id}` reference only. Safe for `member` access.
- `ResolveSecretValue` via Gateway restricted to `admin` on `cluster:global` (was `owner` on org). Internal Orchestrator path remains Istio-only with no OpenFGA check.
- Agent ENVs are never exposed via the Agents Service API to running workloads — assembly-time injection is the only delivery path.

### agynd startup

- `agynd` fetches `GetAgent`, `ListSkills`, `ListInitScripts`, `ListMCPs` from the Gateway at startup.
- `agynd` reads all user-defined ENVs from the process environment (injected by Orchestrator); does not call `ListENVs`.
- MCP server port assignments read from `AGENT_MCP_SERVERS` env var.
- `WORKLOAD_ID` added to agent container env (used for keepalives and span attribution).
- `AGENT_SKILLS` and `AGENT_INIT_SCRIPTS` removed from env table (now fetched via API).

### Per-service authorization sections added

Authorization tables added to services that were missing them: Apps Service, Chat, Expose Service, Notifications, Runners, Threads.
