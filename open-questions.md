# Open Questions

Unresolved architectural decisions. Each item needs discussion before implementation.

---

## Reconciliation Approach

**Context:** The platform is migrating to become k8s-native. A reconciliation loop is needed to converge actual state toward desired state for dynamic resources (agents, channels, workspaces).

**Options to evaluate:**

1. **CRDs + Operators** — Use Kubernetes Custom Resource Definitions and operator pattern. Resources are k8s-native objects. The operator watches CRDs and reconciles.
   - *Pros:* Native k8s tooling (kubectl, RBAC, etcd as store, watch semantics built-in). Ecosystem compatibility.
   - *Cons:* Couples resource model to k8s API. CRD schema evolution can be complex. Requires operator framework expertise. Harder to run outside k8s.

2. **Custom reconciliation loop over own data store** — Application-level reconciliation loop reading desired state from PostgreSQL/Redis and converging.
   - *Pros:* Portable (runs outside k8s). Full control over reconciliation semantics. Simpler to test.
   - *Cons:* Must implement watch/notification, leader election, and consistency guarantees that k8s provides for free.

**Decision:** TBD — Needs dedicated discussion comparing operational complexity, portability requirements, and team expertise.

---

## Runner Split

**Context:** The Runner currently combines workload scheduling (control) and workload execution (data) in a single service. This conflicts with the control/data plane separation.

**Questions:**
- What is the exact boundary between Runner Controller and Runner Worker?
- Does the Runner Controller own workload desired state, or does it read it from the Agents orchestrator?
- How do Controller and Worker communicate? (gRPC between them, or shared state?)

---

## Agent Protocol

**Context:** Most 3rd-party agents are CLI-based. The platform needs a protocol to communicate with agent processes running in containers — providing configuration, connecting MCP tools, and collecting output.

**Options:**

1. **Push (gRPC)** — Platform pushes messages/config to the agent via gRPC. Agent exposes a gRPC server.
   - *Pros:* Immediate delivery. Platform controls flow.
   - *Cons:* Agent must implement a gRPC server. Harder for simple CLI wrappers.

2. **Pull** — Agent pulls pending work from the platform. No exposed methods needed on the agent side.
   - *Pros:* Simpler agent implementation. Agent controls its own pace.
   - *Cons:* Latency (polling interval). Platform needs a queue/inbox per agent.

**Decision:** TBD

---

## Agent Batching Protocol

**Context:** In the simple case, one container is created per agent invocation. For some agents, it may be more efficient to have a single instance process multiple threads (e.g., one agent handles 10 threads).

**Questions:**
- What triggers batch assignment? (Agent type configuration? Runtime heuristic?)
- How does a batched agent receive and route messages for different threads?
- How is isolation maintained between threads in a batched agent?
- What happens when one thread in a batch fails?

---

## Threads Service Extraction

**Context:** Threads is described as a standalone service, but the messaging logic is currently embedded in the platform-server monolith.

**Questions:**
- What data store will Threads use? (Dedicated PostgreSQL schema? Separate database?)
- How does Threads interact with the Agents orchestrator? (Notifications-based? Direct callback? Polling?)
- Does Threads need its own read-status tracking, or is that a concern of the consuming service (e.g., Channels, UI)?

---

## Channel Interface Definition

**Context:** Channels need a formal interface that all implementations (Slack, web app, mobile app) must satisfy.

**Questions:**
- Is the channel interface a gRPC service contract, a container contract, or an SDK/library interface?
- How are channel credentials managed and rotated? (Vault? Teams service? Channel-specific config store?)
- How does the control plane detect that a channel connection is unhealthy and needs reconciliation?

---

## Filesystem Store Migration

**Context:** The graph store currently uses a filesystem-based dataset for persistence. Other services use PostgreSQL and Redis.

**Questions:**
- Will the filesystem store be migrated to PostgreSQL or another store?
- If not, how is it backed up and replicated in a k8s environment (PVCs, object storage)?
