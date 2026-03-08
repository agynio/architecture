# Open Questions

Unresolved architectural decisions requiring discussion.

---

## Reconciliation Approach

**Context:** The platform is migrating to become k8s-native. A reconciliation loop is needed to converge actual state toward desired state for dynamic resources (agents, channels, workspaces).

**Options:**

1. **CRDs + Operators** — Resources as Kubernetes Custom Resource Definitions, reconciled by operators.
   - *Pros:* Native k8s tooling (kubectl, RBAC, etcd, watch semantics). Ecosystem compatibility.
   - *Cons:* Couples resource model to k8s API. CRD schema evolution complexity. Requires operator expertise. Harder outside k8s.

2. **Custom reconciliation loop** — Application-level loop reading desired state from PostgreSQL/Redis.
   - *Pros:* Portable (works outside k8s). Full control. Simpler to test.
   - *Cons:* Must reimplement watch/notification, leader election, consistency guarantees.

**Decision:** TBD — Needs dedicated discussion.

---

## Agent Protocol

**Context:** Most 3rd-party agents are CLI-based. The platform needs a protocol to communicate with agent processes in containers — providing configuration, connecting MCP tools, and collecting output.

**Options:**

1. **Push (gRPC)** — Platform pushes messages/config to the agent. Agent exposes a gRPC server.
   - *Pros:* Immediate delivery. Platform controls flow.
   - *Cons:* Agent must implement a gRPC server. Harder for simple CLI wrappers.

2. **Pull** — Agent pulls pending work from the platform. No exposed methods on agent side.
   - *Pros:* Simpler agent implementation. Agent controls pace.
   - *Cons:* Latency (polling). Platform needs a queue/inbox per agent.

**Decision:** TBD

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
