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

## Threads Service Data Store

**Context:** Threads is being extracted from the monolith into a standalone service.

**Questions:**
- Dedicated PostgreSQL schema or separate database?

---

## Channel Interface Definition

**Context:** Channels need a formal interface that all implementations (Slack, etc.) must satisfy.

**Questions:**
- Is the channel interface a gRPC service contract, a container contract, or an SDK/library interface?
- How are channel credentials managed and rotated?
- How does the control plane detect unhealthy channel connections for reconciliation?

---

## ~~Scheduler Service~~ — Resolved

**Decision:** There is no separate Scheduler service. The [Agents Orchestrator](architecture/agents-orchestrator.md) is the single control plane service that decides what agent workloads should run and when. It reconciles directly with the Runner. A separate MCP reconciliation service will be added later for MCP server discovery — that is a distinct concern from agent scheduling.


---

## OpenZiti: Agent-to-Agent Private Networking

**Context:** Future capability. Agents should be able to expose a port and share it with specific other agents over a private OpenZiti connection. Other agents must not be able to connect. See [OpenZiti Integration — Dynamic Policies](architecture/openziti.md#dynamic-policies-future).

**Questions:**
- How does the agent request port sharing? (Platform API tool? Agent SDK primitive?)
- What is the resource model? (Does the platform define "agent networks" as a team resource, or is each share ad-hoc?)
- Does the binding agent create the OpenZiti service dynamically, or does the Ziti Management service pre-provision it?

---

## OpenZiti: User-to-Agent Direct Connection

**Context:** Future capability. Users should be able to connect directly to a specific agent from their machine via OpenZiti, bypassing the Gateway for certain use cases. The platform will ship a CLI that enrolls the user's machine. See [OpenZiti Integration — Dynamic Policies](architecture/openziti.md#dynamic-policies-future).

**Questions:**
- What is the CLI enrollment UX? (OIDC browser flow → enrollment JWT → local identity file?)
- What protocol does the user use to connect to the agent? (Raw TCP? HTTP? gRPC?)
- How is the direct connection authorized relative to the user's tenant permissions?

---

## Summarization Packaging

**Context:** Summarization is currently embedded in the agent code. As we support multiple agent implementations, summarization logic needs to be reusable.

**Options:**

1. **Part of the agent service** — Each agent implementation bundles its own summarization. Current state.
   - *Pros:* Simple. No external dependencies. Full control per implementation.
   - *Cons:* Duplicated across implementations. Hard to share improvements.

2. **Reusable package** — Shared library (e.g., `@agyn/summarization`) imported by agent implementations.
   - *Pros:* Single source of truth. Each agent can opt in. Language-portable (publish for TS, Go, etc.).
   - *Cons:* Package versioning and compatibility. Must be general enough for different agent architectures.

3. **Standalone service** — Dedicated summarization service with its own gRPC API.
   - *Pros:* Language-agnostic. Centralized. Can be scaled and monitored independently.
   - *Cons:* Network overhead per summarization call. Another service to operate. Latency-sensitive.

**Decision:** TBD

---

## Authorization Model Completeness

**Context:** The authorization approach is decided — OpenFGA with a dedicated Authorization service (see [Authorization](architecture/authz.md)). The initial model covers threads, files, tenant roles, and agent state. The full model needs to cover all resource types and access patterns.

**Questions:**
- What are the complete relation definitions for all resource types (models, workspaces, MCP servers, channels, secrets)?
- How are tracing data permissions modeled? (Tenant-level visibility, or per-agent restriction?)
- How are Notifications subscriptions authorized? (Implicitly via participant membership, or explicit check?)
- What is the relationship tuple cleanup strategy when resources are deleted?

---

## agynd — Agent CLI Protocol

**Context:** [`agynd`](architecture/agynd-cli.md) spawns agent CLIs ([`agn`](architecture/agn-cli.md), 3rd-party CLIs) as child processes and needs to exchange messages with them. The interface between `agynd` and agent CLIs is not yet defined.

**Options:**

1. **Simple shell commands** — `agynd` invokes the agent CLI per message or feeds messages via stdin as plain text, collects responses from stdout.
   - *Pros:* Simplest. Any CLI works out of the box. Easy to test with `cat`, `echo`, etc.
   - *Cons:* Limited structure. Hard to communicate metadata, errors, or multi-part responses.

2. **Structured stdin/stdout protocol** — Newline-delimited JSON or similar structured format over stdin/stdout.
   - *Pros:* Supports metadata, structured errors, streaming. Still simple to implement.
   - *Cons:* 3rd-party CLIs need adaptation or a thin adapter script.

3. **SDK / library** — `agynd` provides a library that agent implementations import, handling communication internally.
   - *Pros:* Type-safe. Can evolve the protocol without breaking the CLI interface.
   - *Cons:* Language-specific. Tighter coupling. 3rd-party CLIs still need a wrapper.

**Decision:** TBD

---

## agn Configuration Structure

**Context:** [`agn`](architecture/agn-cli.md) needs a configuration structure that defines how it operates: LLM endpoint, MCP server connection, skills location, state persistence backend, summarization settings, etc. The configuration format and schema are not yet defined.

**Questions:**
- What is the configuration file format? (YAML, TOML, JSON, environment variables only?)
- What is the directory convention for skills on the filesystem?
- How are LLM endpoint credentials provided? (Config file, environment variable, credential file?)
- How does `agn` discover its MCP server? (Stdio command, socket path, environment variable?)
- What is the schema for state persistence backend selection? (Flag, config field, environment variable?)

---

## agynd Configuration Strategies per Agent

**Context:** [`agynd`](architecture/agynd-cli.md) prepares the runtime environment before spawning an agent CLI. Different agent CLIs expect different configuration conventions — skills in specific directories, environment variables, config files, MCP server setup, etc. `agynd` needs a strategy for each supported agent type.

**Known agent CLIs requiring strategies:**

| Agent CLI | Configuration Needs |
|-----------|-------------------|
| **[`agn`](architecture/agn-cli.md)** | Skills directory, LLM endpoint, MCP server, state persistence backend. Format TBD (see [agn Configuration Structure](#agn-configuration-structure)) |
| **Claude Code** | `CLAUDE.md` files in workspace, environment variables for API keys, MCP server config via `claude_desktop_config.json` or CLI flags |
| **Codex CLI** | `AGENTS.md` or instructions file, environment variables, MCP server configuration |

**Questions:**
- How does `agynd` know which strategy to apply? (Agent resource field? Image convention? Explicit strategy name in configuration?)
- Is the strategy a hard-coded mapping per agent type, or a pluggable adapter?
- How are agent-type-specific configuration conventions discovered and maintained as 3rd-party CLIs evolve?

**Decision:** TBD
