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

## Scheduler Service

**Context:** Currently the Runner is purely data plane (executes workloads). A separate **Scheduler** service may be needed in the control plane to decide *what* and *when* to run.

**Questions:**
- Is the Scheduler the same as the Agents orchestrator, or a lower-level service that the orchestrator delegates to?
- Does the Scheduler manage only agent workloads, or also workspace lifecycle and TTL?
- What is the scheduling interface? (Request/response? Queue-based? Watch-based?)


---

## OpenZiti Integration

**Context:** Agent containers, channels, and runners will use OpenZiti for network-level identity and mTLS. This replaces application-level tokens for authentication.

**Questions:**
- How does Runner integrate with the OpenZiti Controller API? (SDK? CLI? REST?)
- What is the identity lifecycle for agent containers? (create on start, delete on stop — what about crashes/orphans?)
- How are OpenZiti service policies managed? (Per-agent? Per-tenant? Static set of allowed services?)
- How does Gateway extract identity from OpenZiti mTLS connections? (Istio integration? Direct cert inspection?)
- Can OpenZiti identities carry tenant metadata, or must the platform maintain a separate identity-to-tenant mapping?

---

## Internal Service Authentication

**Context:** Internal service-to-service communication is currently unauthenticated. The target is Istio mTLS with strict PeerAuthentication policies.

**Questions:**
- How is end-user identity propagated across service boundaries? (gRPC metadata? Header convention?)
- Should services verify the calling service identity (e.g., only Gateway can call Chat), or is network policy sufficient?

---

## External Runner Connectivity

**Context:** Users will connect their own runners in external networks to the platform using OpenZiti.

**Questions:**
- How does an external runner enroll with the platform's OpenZiti network?
- How are agent containers started by external runners given OpenZiti identities? (Runner requests identity from platform's OpenZiti Controller?)
- What services can agent containers on external runners access? (Only Gateway? Direct access to Threads, Files?)
- How does the platform verify that an external runner is authorized to start workloads for a given tenant?

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
