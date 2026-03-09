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
- How does Threads interact with Agents orchestrator? (Notifications-based? Direct callback? Polling?)
- Does Threads own read-status tracking, or is that a consumer concern?

---

## Channel Interface Definition

**Context:** Channels need a formal interface that all implementations (Slack, web app, mobile app) must satisfy.

**Questions:**
- Is the channel interface a gRPC service contract, a container contract, or an SDK/library interface?
- How are channel credentials managed and rotated?
- How does the control plane detect unhealthy channel connections for reconciliation?

---

## Filesystem Store Migration

**Context:** Graph definitions use a filesystem-based dataset. Other services use PostgreSQL and Redis.

**Questions:**
- Will the filesystem store be migrated to PostgreSQL or another store?
- If not, how is it backed up and replicated in k8s (PVCs, object storage)?

---

## Scheduler Service

**Context:** Currently the Runner is purely data plane (executes workloads). A separate **Scheduler** service may be needed in the control plane to decide *what* and *when* to run.

**Questions:**
- Is the Scheduler the same as the Agents orchestrator, or a lower-level service that the orchestrator delegates to?
- Does the Scheduler manage only agent workloads, or also workspace lifecycle and TTL?
- What is the scheduling interface? (Request/response? Queue-based? Watch-based?)

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

## Context Size Measurement with Media

**Context:** The current summarization trigger estimates token count from text length (`text.length / 4`). With media files (images, PDFs, documents), this heuristic is invalid — the LLM provider determines actual token consumption based on internal processing (e.g., image tile count, extracted text length).

**Planned direction:** Use the `usage.input_tokens` value returned by the LLM provider's response to measure actual context size after each call, rather than estimating tokens before the call.

**Questions:**
- How does the summarization trigger work if context size is only known after the LLM call? Currently, summarization runs *before* the LLM call to ensure context fits the budget. With post-call measurement, the first call may exceed the budget and need a retry after summarization.
- Should we adopt a two-phase approach: estimate conservatively before the call, then refine with actual usage after?
- How should media files interact with the summarization fold? Images and files cannot be "summarized" into text the same way messages can. Should they be dropped, kept verbatim, or replaced with text descriptions?
- Does the `summarizationKeepTokens` budget include media token costs, or is it text-only?
