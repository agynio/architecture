# OpenZiti Integration

## Overview

The platform uses [OpenZiti](https://openziti.io/) as its overlay network layer, providing:

- **Network-level identity** — Every agent, channel, runner, and orchestrator has a unique x509 certificate issued by the OpenZiti Controller.
- **mTLS transport** — All cross-boundary traffic uses mutual TLS. Identity is in the certificate, not in application-level tokens.
- **Service-level access control** — OpenZiti service policies determine which identities can reach which services (ABAC model using role attributes).

This document covers the **implementation** of OpenZiti integration: which service manages identities, how policies are structured, how the Gateway extracts identity, and how the Orchestrator handles identity lifecycle. For the conceptual design (identity types, enrollment flows, two-network-layer architecture), see [Authentication](authn.md).

## SDK Embedding

**Infrastructure services** that participate in both the Istio mesh and the OpenZiti overlay embed the **OpenZiti Go SDK** ([`openziti/sdk-golang`](https://github.com/openziti/sdk-golang)) directly in the application process. This avoids deploying an OpenZiti sidecar alongside the Istio sidecar, which would create conflicts over outbound traffic routing.

**Agent pods** are not in the Istio mesh. They use a **Ziti sidecar container** within the pod that enrolls the agent's OpenZiti identity and exposes OpenZiti services as local network addresses accessible to all processes in the pod. See [Agent Access Scope](#agent-access-scope).

The OpenZiti Go SDK implements Go's standard `net.Listener` and `net.Conn` interfaces. A gRPC server accepts connections from an OpenZiti listener the same way it accepts from a TCP listener. A gRPC client dials through an OpenZiti context the same way it dials a TCP address.

| Service | SDK Role | How |
|---------|----------|-----|
| **Runner** | Bind (server) | `zitiContext.Listen("runner")` → `grpcServer.Serve(listener)` |
| **Agents Orchestrator** | Dial (client) | `zitiContext.Dial("runner")` → gRPC client connection |
| **Gateway** | Bind (server) | `zitiContext.ListenWithOptions("gateway", ...)` → accept connections |
| **LLM Proxy** | Bind (server) | `zitiContext.ListenWithOptions("llm-proxy", ...)` → accept connections |

Agent pods do not embed the OpenZiti SDK. Instead, a **Ziti sidecar** runs alongside the agent process within the same pod. See [Agent Access Scope](#agent-access-scope).

Each of these services obtains its OpenZiti identity at runtime via self-enrollment through the [Ziti Management](#ziti-management-service) service. The identity (x509 certificate and private key) is held on the pod's ephemeral disk for the lifetime of the process. If the pod restarts, it requests a new identity. See [Service Identity Self-Enrollment](#service-identity-self-enrollment) and [Authentication](authn.md#enrollment).

## Ziti Management Service

All interactions with the OpenZiti Controller's Edge Management API are encapsulated in a dedicated **Ziti Management** service.

| Aspect | Detail |
|--------|--------|
| **Plane** | Data — executes operations it is told to perform; runs identity lease GC |
| **Consumers** | Agents Orchestrator (agent identity lifecycle), Gateway (identity resolution), Orchestrator / Runner / Gateway (service identity self-enrollment) |
| **External dependency** | OpenZiti Edge Management API (`/edge/management/v1/`) via the generated Go client (`openziti/edge-api`, package `rest_management_api_client`) |
| **Access to Controller** | Via Istio (in-cluster) — avoids circular dependency of using OpenZiti to manage OpenZiti |
| **Authentication** | Certificate-based auth using a long-lived infrastructure credential provisioned at deployment |
| **Data store** | PostgreSQL — stores identity mappings (OpenZiti identity ID ↔ platform identity ID, identity type) and lease timestamps for service identities |

### Why a Separate Service

- The Orchestrator reconciles agent desired state. It calls Ziti Management as a side effect, same as it calls Runner. Isolating OpenZiti logic keeps the Orchestrator focused on lifecycle decisions.
- The Gateway needs identity resolution on the request hot path. It calls Ziti Management, which reads from PostgreSQL — no dependency on the OpenZiti Controller for every request.
- If OpenZiti's API changes, only one service changes.
- Infrastructure services (Orchestrator, Runner, Gateway) self-enroll through Ziti Management at startup, keeping all OpenZiti Controller interactions in one place.
- Future capabilities (user enrollment for CLI, agent-to-agent port sharing) will also route through this service.

### API Surface

| RPC | Caller | Description |
|-----|--------|-------------|
| `CreateAgentIdentity` | Orchestrator | Create an OpenZiti identity for an agent, return enrollment JWT |
| `DeleteIdentity` | Orchestrator | Delete an OpenZiti identity and its platform mapping |
| `ListManagedIdentities` | Orchestrator | List all identities managed by the platform (for reconciliation) |
| `ResolveIdentity` | Gateway | Map an OpenZiti identity ID to platform identity (identity_id, identity_type) |
| `RequestServiceIdentity` | Orchestrator, Runner, Gateway | Create and enroll an OpenZiti identity for the calling service, return enrolled identity (cert + key) |
| `ExtendIdentityLease` | Orchestrator, Runner, Gateway | Extend the lease on a service identity |

### OpenZiti Controller Operations

| Operation | API Endpoint | When |
|-----------|-------------|------|
| Create identity | `POST /edge/management/v1/identities` | Agent start, service self-enrollment, runner/channel enrollment |
| Enroll identity | Controller enrollment endpoint | Service identity self-enrollment (Ziti Management enrolls on behalf of the caller) |
| Delete identity | `DELETE /edge/management/v1/identities/{id}` | Agent stop, identity cleanup, lease GC |
| List identities | `GET /edge/management/v1/identities?filter=...` | Reconciliation, lease GC |
| Update role attributes | `PATCH /edge/management/v1/identities/{id}` | Future: dynamic policy changes |
| Create service | `POST /edge/management/v1/services` | Future: agent-to-agent networking |
| Create service policy | `POST /edge/management/v1/service-policies` | Future: dynamic access grants |
| Delete service / policy | `DELETE /edge/management/v1/{type}/{id}` | Future: revoke dynamic access |

### Identity Lease GC

Ziti Management runs a background loop that garbage-collects expired service identities:

1. Each pass: query PostgreSQL for service identities whose lease has expired (no `ExtendIdentityLease` call within the configured TTL).
2. Delete the expired OpenZiti identity on the Controller via `DELETE /edge/management/v1/identities/{id}`.
3. Remove the identity record from PostgreSQL.

This is a separate concern from the Orchestrator's agent identity reconciliation. The Orchestrator reconciles agent identities against running workloads. Ziti Management GC handles service identities (Orchestrator, Runner, Gateway) based purely on lease expiry — it does not query Runner or any other service.

Expired identities are inert — the pod that held the enrolled certificate has already stopped (otherwise it would have extended the lease). GC is important for hygiene and OpenZiti Controller resource limits.

## Identity Management

### Who Manages Identities

Two identity lifecycle patterns coexist:

- **Agent identities** — managed by the **Agents Orchestrator**. The Orchestrator creates and deletes agent identities via Ziti Management as part of its reconciliation loop. The Runner is not involved in identity management — it receives the enrollment JWT as opaque configuration and passes it to the agent pod's Ziti sidecar container.
- **Service identities** — self-managed by each service (Orchestrator, Runner, Gateway). Each pod requests its own identity from Ziti Management at startup and extends the lease on a timer. See [Service Identity Self-Enrollment](#service-identity-self-enrollment).

The agent identity pattern follows directly from the [Control Plane & Data Plane](control-data-plane.md) classification: the Orchestrator is control plane (decides what should exist), the Runner is data plane (executes what it is told).

### Agent Identity Lifecycle

```mermaid
sequenceDiagram
    participant O as Agents Orchestrator
    participant ZM as Ziti Management
    participant ZC as OpenZiti Controller
    participant R as Runner
    participant A as Agent Pod

    Note over O: Reconciliation: thread needs agent
    O->>ZM: CreateAgentIdentity(agentId)
    ZM->>ZC: POST /identities (name, type, roleAttributes, externalId, enrollment.ott)
    ZC->>ZM: Identity ID + enrollment JWT
    ZM->>ZM: Store mapping in PostgreSQL
    ZM->>O: enrollmentJWT, openZitiIdentityId
    O->>O: Store workload ↔ identity mapping in PostgreSQL
    O->>R: StartWorkload(config, enrollmentJWT)
    R->>A: Start pod with enrollment JWT
    Note over A: Ziti sidecar enrolls (exchanges JWT for x509 cert)
    A->>A: All pod processes reach platform APIs via sidecar-provided local addresses

    Note over O: Reconciliation: agent idle
    O->>R: StopWorkload(workloadId)
    R->>A: Stop pod
    O->>ZM: DeleteIdentity(openZitiIdentityId)
    ZM->>ZC: DELETE /identities/{id}
    ZM->>ZM: Remove mapping from PostgreSQL
```

### Identity Creation Request

When Ziti Management creates an agent identity, it sends:

```json
{
  "name": "agent-<agentId>-<shortUuid>",
  "type": "Device",
  "isAdmin": false,
  "roleAttributes": ["agents", "agent-<agentId>"],
  "externalId": "<platformIdentityId>",
  "enrollment": { "ott": true }
}
```

| Field | Purpose |
|-------|---------|
| `name` | Human-readable, unique per identity. Includes agent ID for debugging |
| `type` | `Device` — represents a non-human endpoint |
| `roleAttributes` | Tags for ABAC policy matching. `agents` for static policies, `agent-<agentId>` for future per-agent policies |
| `externalId` | Platform identity UUID — the link between OpenZiti identity and platform identity |
| `enrollment.ott` | One-time-token enrollment. Controller generates a JWT valid for 24 hours |

### Orphan Reconciliation

The Orchestrator's existing reconciliation loop handles orphaned identities (from Runner crashes, pod crashes, or Orchestrator restarts):

1. Each reconciliation pass: Orchestrator calls `ZitiManagement.ListManagedIdentities()`.
2. Compares against active workloads from `Runner.FindWorkloadsByLabels()`.
3. Deletes OpenZiti identities that have no matching running workload via `ZitiManagement.DeleteIdentity()`.

This is the same reconciliation pattern used for agent workloads — no new mechanism needed. An orphaned identity with no running pod is inert (it can't be used because the enrollment JWT has expired or the enrolled certificate is inside a stopped pod), but cleanup is important for hygiene and OpenZiti Controller resource limits.

## Service Identity Self-Enrollment

Infrastructure services that participate in the OpenZiti overlay (Orchestrator, Runner, Gateway) obtain their OpenZiti identities at runtime by self-enrolling through Ziti Management.

### Design

- **Ephemeral identities** — each pod gets a new OpenZiti identity on startup. The identity is not shared across replicas or preserved across restarts.
- **In-process enrollment** — the service calls Ziti Management over Istio (in-cluster gRPC). Ziti Management creates the identity on the Controller, enrolls it, and returns the enrolled identity (certificate + key) to the caller.
- **Ephemeral disk storage** — the enrolled identity is written to ephemeral disk (emptyDir or tmpfs) so the OpenZiti SDK can load it as a file. The identity does not survive pod restart.
- **Lease-based lifecycle** — the service extends its lease on a timer. Ziti Management garbage-collects identities whose lease expires. This makes the system eventually consistent — no coordination required between pods and Ziti Management beyond the heartbeat.

### Flow

```mermaid
sequenceDiagram
    participant P as Service Pod
    participant ZM as Ziti Management
    participant ZC as OpenZiti Controller

    Note over P: Pod starts
    P->>ZM: RequestServiceIdentity(roleAttributes, serviceType)
    ZM->>ZC: POST /identities (type: Device, roleAttributes, enrollment.ott)
    ZC->>ZM: Identity ID + enrollment JWT
    ZM->>ZC: Enroll (exchange JWT for x509 cert + key)
    ZC->>ZM: Certificate + private key
    ZM->>ZM: Store identity record + lease timestamp in PostgreSQL
    ZM->>P: Enrolled identity (cert + key)
    P->>P: Write identity to ephemeral disk
    P->>P: Load identity via OpenZiti SDK

    loop Every lease interval
        P->>ZM: ExtendIdentityLease(identityId)
        ZM->>ZM: Update lease timestamp in PostgreSQL
    end

    Note over P: Pod stops (crash, scale-down, restart)
    Note over ZM: Lease expires → GC deletes identity
```

### Service-Specific Behavior

| Service | Role Attributes | After enrollment |
|---------|----------------|-----------------|
| **Internal Runner** | `["runners"]` | `zitiContext.Listen("runner")` — binds the `runner` service |
| **Agents Orchestrator** | `["orchestrators"]` | `zitiContext.Dial("runner")` — dials runners |
| **Gateway** | `["gateway-hosts"]` | `zitiContext.ListenWithOptions("gateway", ...)` — binds the `gateway` service |
| **LLM Proxy** | `["llm-proxy-hosts"]` | `zitiContext.ListenWithOptions("llm-proxy", ...)` — binds the `llm-proxy` service |

## Service Policies

OpenZiti uses an ABAC (Attribute-Based Access Control) model. Service policies match identities to services using **role attributes** (string tags). There are two policy types: **Dial** (connect to a service) and **Bind** (host a service).

### Role Attributes

| Identity Type | Role Attributes |
|---|---|
| Agent pod (Ziti sidecar) | `["agents", "agent-<agentId>"]` |
| Channel | `["channels"]` |
| Runner | `["runners"]` |
| Orchestrator | `["orchestrators"]` |
| LLM Proxy | `["llm-proxy-hosts"]` |

The `agent-<agentId>` attribute is assigned at creation time for future use in per-agent policies. It has no effect until matching service policies are created.

### Static Policies

Defined once at infrastructure provisioning (Terraform / bootstrap scripts). These cover baseline connectivity:

| Policy | Type | Identity Roles | Service Roles | Purpose |
|--------|------|---------------|---------------|---------|
| `agents-dial-gateway` | Dial | `#agents` | `@gateway` | All agents can reach Gateway |
| `channels-dial-gateway` | Dial | `#channels` | `@gateway` | All channels can reach Gateway |
| `orchestrators-dial-runners` | Dial | `#orchestrators` | `@runner` | Orchestrator can reach Runners |
| `gateway-bind` | Bind | `#gateway-hosts` | `@gateway` | Gateway hosts the `gateway` service |
| `runners-bind` | Bind | `#runners` | `@runner` | Runners host the `runner` service |
| `agents-dial-llm-proxy` | Dial | `#agents` | `@llm-proxy` | All agents can reach LLM Proxy |
| `llm-proxy-bind` | Bind | `#llm-proxy-hosts` | `@llm-proxy` | LLM Proxy hosts the `llm-proxy` service |

Edge router policies: `#all` identities → `#all` edge routers (no router-level segmentation needed).

Static policies, services, and edge router policies are provisioned by Terraform at bootstrap. Identity provisioning is handled separately via [self-enrollment](#service-identity-self-enrollment) — policies match identities by role attributes, not by specific identity references.

### Dynamic Policies (Future)

The static policies above are sufficient for the current architecture. Future capabilities will require dynamic, per-agent or per-user policies managed at runtime by Ziti Management:

- **User-to-agent direct connection** — A user enrolls their machine via a CLI tool and connects directly to a specific agent over OpenZiti. Requires creating a per-agent service and scoped Dial/Bind policies at runtime.
- **Agent-to-agent private networking** — An agent exposes a port that only specific other agents can connect to. Requires creating a per-share service and scoped Dial/Bind policies at runtime.

OpenZiti supports this natively:

- **Role attributes are mutable.** `PATCH /edge/management/v1/identities/{id}` can update `roleAttributes` on an already-enrolled, connected identity. No re-enrollment needed.
- **Policy changes take effect immediately.** New service policies grant (or revoke) access for matching identities in real time.
- **Revocation closes live connections.** Removing a policy tears down existing connections, not just prevents new ones.

These capabilities mean dynamic policies can be applied to already-running agents without restart. The Ziti Management service API will be extended with RPCs for port sharing and user enrollment when these features are implemented.

## Gateway Identity Extraction

The Gateway binds an OpenZiti service and extracts the caller's identity from each incoming connection.

### Flow

```mermaid
sequenceDiagram
    participant A as Agent / Channel
    participant GW as Gateway
    participant ZM as Ziti Management

    A->>GW: Connect via OpenZiti mTLS
    GW->>GW: AcceptEdge() → edge.Conn
    GW->>GW: conn.SourceIdentifier() → OpenZiti identity ID
    GW->>ZM: ResolveIdentity(openZitiIdentityId)
    ZM->>ZM: Read mapping from PostgreSQL
    ZM->>GW: (identity_id, identity_type)
    GW->>GW: Inject into gRPC metadata
```

### Implementation

1. Gateway starts, obtains its OpenZiti identity via [self-enrollment](#service-identity-self-enrollment), calls `ctx.ListenWithOptions("gateway", ...)`.
2. On each incoming connection: `conn, _ := listener.AcceptEdge()`.
3. `conn.SourceIdentifier()` returns the dialing identity's name and ID. This is set by the OpenZiti router at circuit creation time — it is not self-asserted by the client and is therefore trustworthy.
4. Gateway calls `ZitiManagement.ResolveIdentity(openZitiIdentityId)`. Ziti Management reads the mapping from PostgreSQL (written when the identity was created). Returns `(identity_id, identity_type)`.
5. Gateway injects these values into gRPC metadata for downstream services — same format as OIDC-authenticated user requests.

### Caching

Ziti Management can cache resolved identities in-memory with a short TTL. Identity mappings are immutable for the lifetime of an OpenZiti identity — the cache only needs invalidation when an identity is deleted.

## Identity Resolution

The canonical resolution path for OpenZiti identities is:

```
OpenZiti identity ID → ZitiManagement.ResolveIdentity() → PostgreSQL → (identity_id, identity_type)
```

Organization context is not part of the OpenZiti identity. Services that need organization context accept `organization_id` as a request parameter. See [Organizations — Request Flow](organizations.md#request-flow).

## Internal Identity Propagation

After Gateway authenticates a request (OIDC for users, OpenZiti for agents/channels/runners), it injects the resolved identity into gRPC metadata:

| Metadata Key | Type | Description |
|-------------|------|-------------|
| `x-identity-id` | string (UUID) | Platform identity ID |
| `x-identity-type` | string | `user`, `agent`, `channel`, `runner` |

All internal services read these keys from incoming gRPC metadata. Services trust these values because:

- Internal traffic is Istio mTLS (only verified cluster services can call each other).
- Istio `AuthorizationPolicy` restricts which ServiceAccounts can call which services.
- The Gateway is the only service that sets these headers; internal services read and forward them.

## Runner Provisioning

Runners connect to the OpenZiti overlay to bind the `runner` service and receive work from the Orchestrator. Internal and external runners use the same protocol (OpenZiti mTLS) but differ in how their OpenZiti identity is provisioned.

### Internal Runners

Internal runners are deployed as part of the platform. Their OpenZiti identity is obtained at runtime via [self-enrollment](#service-identity-self-enrollment):

```mermaid
sequenceDiagram
    participant R as Runner Pod
    participant ZM as Ziti Management
    participant ZC as OpenZiti Controller

    R->>ZM: RequestServiceIdentity(roleAttributes: ["runners"])
    ZM->>ZC: POST /identities (type: Device, roleAttributes: ["runners"], enrollment.ott)
    ZC->>ZM: Identity ID + enrollment JWT
    ZM->>ZC: Enroll (exchange JWT for x509 cert + key)
    ZC->>ZM: Certificate + private key
    ZM->>ZM: Store identity + lease in PostgreSQL
    ZM->>R: Enrolled identity (cert + key)
    R->>R: Write identity to ephemeral disk
    R->>R: Load identity, zitiContext.Listen("runner")
    Note over R: Accepting gRPC connections via OpenZiti

    loop Every lease interval
        R->>ZM: ExtendIdentityLease(identityId)
        ZM->>ZM: Update lease timestamp in PostgreSQL
    end
```

1. On startup, the runner calls `ZitiManagement.RequestServiceIdentity()` with `roleAttributes: ["runners"]`.
2. Ziti Management creates the identity on the Controller, enrolls it, stores the mapping and lease timestamp in PostgreSQL, and returns the enrolled identity (certificate + key).
3. The runner writes the identity to ephemeral disk and loads it via the OpenZiti SDK.
4. The runner binds the `runner` OpenZiti service.
5. On a timer, the runner calls `ZitiManagement.ExtendIdentityLease()` to keep the identity alive.
6. If the pod restarts, it requests a new identity. The old identity's lease expires and is cleaned up by Ziti Management's [GC loop](#identity-lease-gc).

### External Runners

External runners are provisioned by an operator using a service token:

1. Admin registers a runner via the [Runners](runners.md) service. The system generates a service token.
2. Operator configures the external runner with the service token.
3. Runner presents the token to the platform enrollment endpoint.
4. Platform validates the token → calls `ZitiManagement.CreateRunnerIdentity(runnerId)`.
5. Ziti Management creates an OpenZiti identity with `roleAttributes: ["runners"]`, returns enrollment JWT.
6. Platform returns the JWT to the Runner. Runner enrolls with the OpenZiti Controller.
7. Runner stores the enrolled identity locally and loads it on subsequent starts.
8. All subsequent communication uses mTLS. The service token is no longer needed.

### Uniform Protocol

Both internal and external runners bind the same `runner` OpenZiti service and are indistinguishable from the Orchestrator's perspective. The Orchestrator dials runners via `zitiContext.Dial("runner")` — it does not know or care whether the runner is in-cluster or remote. This eliminates protocol branching in the Orchestrator.

## Agent Access Scope

Agents connect to the **Gateway** and the **[LLM Proxy](llm-proxy.md)**, regardless of runner location (internal or external). The static service policies `agents-dial-gateway` and `agents-dial-llm-proxy` grant all agents Dial access to the `gateway` and `llm-proxy` services respectively. No other OpenZiti services are dialable by agents.

Agent pods run a **Ziti sidecar container** that enrolls the agent's OpenZiti identity and makes OpenZiti services available as local network addresses within the pod. This means `agynd`, agent CLI subprocesses (Codex CLI, Claude Code, `agn`), and any other process in the pod can reach Gateway and LLM Proxy as regular network calls — no embedded SDK required. The sidecar handles enrollment, mTLS termination, and identity lifecycle within the pod.

| Connection | Layer |
|------------|-------|
| Agent → Gateway | OpenZiti |
| Agent → LLM Proxy | OpenZiti |
| Gateway → internal services | Istio |
| LLM Proxy → internal services | Istio |

The Gateway routes agent requests to internal services (Threads, Files, etc.) via Istio. The LLM Proxy handles LLM API calls — it resolves models via the LLM service and forwards requests to external providers. Agents never connect directly to other internal services.

## OpenZiti Identities Summary

| Identity | Lifecycle | Provisioning | Calls via OpenZiti |
|----------|-----------|--------------|--------------------|
| Agents Orchestrator | Ephemeral (per pod) | Self-enrollment via Ziti Management | Runner (dial) |
| Internal Runner | Ephemeral (per pod) | Self-enrollment via Ziti Management | — (binds `runner` service) |
| External Runner | Persistent (enrolled via service token) | Operator (service token) | — (binds `runner` service) |
| Agent pod (Ziti sidecar) | Ephemeral (per pod) | Orchestrator via Ziti Management | Gateway (dial), LLM Proxy (dial) |
| Channel | Persistent (enrolled via service token) | Operator (service token) | Gateway (dial) |
| Ziti Management | N/A — no OpenZiti network identity | Controller API credential (Terraform) | OpenZiti Controller (via Istio, not overlay) |
| Gateway | Ephemeral (per pod) | Self-enrollment via Ziti Management | — (binds `gateway` service) |
| LLM Proxy | Ephemeral (per pod) | Self-enrollment via Ziti Management | — (binds `llm-proxy` service) |

## App Identity Lifecycle

[Apps](apps.md) connect to the OpenZiti overlay to both bind their own service (receive commands from Gateway) and dial the Gateway (call platform APIs). App identities are created by the [Apps Service](apps-service.md) via Ziti Management during app registration.

### Flow

```mermaid
sequenceDiagram
    participant Admin as Cluster Admin
    participant AS as Apps Service
    participant App as App Process
    participant EP as Enrollment Endpoint
    participant ZM as Ziti Management
    participant ZC as OpenZiti Controller

    Note over Admin: Register app
    Admin->>AS: RegisterApp(slug, name)
    AS->>AS: Generate service token, store app record
    AS-->>Admin: App record + service token

    Note over App: App deployed with service token
    App->>EP: Present service token
    EP->>EP: Validate token (hash lookup via Apps Service)
    EP->>ZM: CreateAppIdentity(appId, roleAttributes: ["apps"])
    ZM->>ZC: POST /identities (type: Device, roleAttributes: ["apps"], enrollment.ott)
    ZC->>ZM: Identity ID + enrollment JWT
    ZM->>ZC: Enroll (exchange JWT for x509 cert + key)
    ZC->>ZM: Certificate + private key
    ZM->>ZM: Store mapping in PostgreSQL
    ZM->>EP: Enrolled identity (cert + key)
    EP->>App: Enrolled identity (cert + key)
    App->>App: Bind own service (e.g., "app-reminders")
    App->>App: Dial Gateway for platform APIs
```

This follows the same pattern as [external runner enrollment](#runner-provisioning). The service token is long-lived — if the app restarts, it re-enrolls and receives a new OpenZiti identity.

### Identity Creation Request

```json
{
  "name": "app-<slug>-<shortUuid>",
  "type": "Device",
  "isAdmin": false,
  "roleAttributes": ["apps"],
  "externalId": "<platformIdentityId>",
  "enrollment": { "ott": true }
}
```

`Device` is the standard OpenZiti type for non-human identities — same as agents and runners.

### Ziti Management API Addition

| RPC | Caller | Description |
|-----|--------|-------------|
| `CreateAppIdentity` | Enrollment endpoint | Create and enroll an OpenZiti identity for an app, return enrolled identity (cert + key) |

This follows the same pattern as external runner enrollment — Ziti Management creates the identity, enrolls it on behalf of the caller, and returns the enrolled credentials.

### OpenZiti Service per App

Each app binds its own OpenZiti service (named `app-{slug}`). The service is created at registration time:

| Operation | When |
|-----------|------|
| `POST /edge/management/v1/services` | App registration — creates the app's service (e.g., `app-reminders`) with `roleAttributes: ["app-services"]` |
| `DELETE /edge/management/v1/services/{id}` | App deletion — removes the service |

### Updated Static Policies

Three new static policies are added at bootstrap:

| Policy | Type | Identity Roles | Service Roles | Purpose |
|--------|------|---------------|---------------|---------|
| `apps-dial-gateway` | Dial | `#apps` | `@gateway` | Apps can reach Gateway |
| `apps-bind` | Bind | `#apps` | `#app-services` | Apps host their own services |
| `gateway-dial-apps` | Dial | `#gateway-hosts` | `#app-services` | Gateway can reach apps |

### Updated Role Attributes Table

| Identity Type | Role Attributes |
|---|---|
| Agent pod (Ziti sidecar) | `["agents", "agent-<agentId>"]` |
| Channel | `["channels"]` |
| Runner | `["runners"]` |
| Orchestrator | `["orchestrators"]` |
| App | `["apps"]` |

### Updated Identities Summary

| Identity | Lifecycle | Provisioning | Calls via OpenZiti |
|----------|-----------|--------------|--------------------|
| App | Persistent (enrolled via enrollment token) | Apps Service via Ziti Management | Gateway (dial) + own service (bind) |
