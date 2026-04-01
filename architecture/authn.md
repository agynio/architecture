# Authentication

## Overview

The platform authenticates four types of identities. Each identity type has its own authentication mechanism, but all resolve to the same internal representation: an `identity_id` and `identity_type`.

## Identity Types

| Type | Description | Authentication Method |
|------|-------------|----------------------|
| **User** | Human operator using web/mobile app | OIDC or [API token](api-tokens.md) |
| **Agent** | Agent pod calling platform APIs | OpenZiti (network identity) |
| **Runner** | Runner executing workloads | OpenZiti (network identity) |
| **App** | [App](apps.md) interacting with threads | OpenZiti (network identity) |

All identity types are represented uniformly as `identity:<identity_id>` in the [authorization model](authz.md). See [Identity](identity.md) for the central identity registry and [Users](users.md) for user-specific details.

## Internal Identity

After authentication, every request carries a resolved identity in its context:

| Field | Type | Description |
|-------|------|-------------|
| `identity_id` | string (UUID) | Unique identity identifier |
| `identity_type` | enum | `user`, `agent`, `runner`, `app` |

Downstream services receive identity context via gRPC metadata. Services use `identity_id` for attribution (e.g., message sender). Organization context is passed as a request parameter where needed — see [Organizations — Request Flow](organizations.md#request-flow).

The `identity_type` indicates the authentication mechanism and profile source (e.g., OIDC users have profiles in [Users](users.md), agents in [Agents](agents-service.md)). Authorization is determined by [relationships](authz.md), not by type.

## User Authentication (OIDC)

Users authenticate via a system-wide OIDC-compliant identity provider. The web app (chat-app) is a single-page application (SPA) that implements the OIDC Authorization Code flow with PKCE. The [Users](users.md) service manages user identity records and profiles.

### Flow

```mermaid
sequenceDiagram
    participant App as chat-app (SPA)
    participant IdP as External IdP
    participant GW as Gateway
    participant US as Users
    participant IS as Identity

    App->>IdP: Redirect to IdP (PKCE: code_challenge)
    IdP->>App: Authorization code
    App->>IdP: Exchange code for tokens (code_verifier)
    IdP->>App: access_token (+ id_token)
    App->>GW: API request (Bearer access_token)
    GW->>GW: Validate JWT signature (IdP JWKS), extract sub
    GW->>US: ResolveUser(oidc_subject)
    US-->>GW: identity_id (or not found)
    alt User not found (first login)
        GW->>IdP: GET /userinfo (Bearer access_token)
        IdP-->>GW: User claims (name, email, picture)
        GW->>US: CreateUser(oidc_subject, profile from claims)
        US->>IS: RegisterIdentity(identity_id, "user")
        US-->>GW: identity_id
    end
    GW->>GW: Propagate identity_id in gRPC metadata
```

The SPA performs the full OIDC flow in the browser. The Gateway receives the resulting `access_token` as a Bearer token on each API request.

**On every request:** The Gateway validates the `access_token` JWT signature against the IdP's JWKS endpoint and extracts the `sub` claim. It then calls `Users.ResolveUser(sub)` to map the OIDC subject to a platform `identity_id`.

**On first login (user not found):** The Gateway calls the IdP's standard [UserInfo endpoint](https://openid.net/specs/openid-connect-core-1_0.html#UserInfo) with the `access_token` to retrieve profile claims (name, email, picture). It then calls `Users.CreateUser(sub, profile)` to provision a new user record. The Users service registers the identity in the [Identity](identity.md) service. The UserInfo endpoint is called **once per user lifetime** — only during provisioning.

Organization context is not validated at the Gateway level. Services that need organization context accept `organization_id` as a request parameter, and the [authorization model](authz.md) enforces access. See [Organizations — Request Flow](organizations.md#request-flow).

### Configuration

The OIDC provider is configured system-wide. Because the SPA is a public client using PKCE, there is no client secret.

| Field | Type | Description |
|-------|------|-------------|
| `issuer` | string | OIDC issuer URL. Used for [OIDC Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html) (`{issuer}/.well-known/openid-configuration`) to resolve the `jwks_uri` for token signature verification |
| `client_id` | string | OAuth2 client ID |

## Network Identity (OpenZiti)

Agents, Runners, Apps, and the Agents Orchestrator authenticate via **OpenZiti** network-level identity. Each receives a unique x509 certificate from the OpenZiti Controller. All API communication uses mTLS over the OpenZiti overlay.

### Enrollment

Non-user identities bootstrap onto the OpenZiti network through one of three paths:

**Self-enrolled service identities** (Orchestrator, Gateway, LLM Proxy) request their identity from the [Ziti Management](openziti.md) service at pod startup. Ziti Management creates the identity on the OpenZiti Controller, enrolls it, and returns the enrolled identity (certificate + key). The pod writes the identity to ephemeral disk and extends a lease on a timer. If the pod restarts, it requests a new identity — the old one is garbage-collected by Ziti Management when its lease expires. See [OpenZiti Integration — Service Identity Self-Enrollment](openziti.md#service-identity-self-enrollment).

**Service-token-provisioned identities** (Runners, Apps) use a service token flow. An admin registers the resource in the platform (via Terraform provider or CLI), receives a service token, and deploys the service with it. On startup, the service presents the token to the platform's enrollment endpoint, which creates an OpenZiti identity, enrolls it, and returns the enrolled identity (certificate + key). The service token is long-lived and reusable — if the service restarts, it re-enrolls and receives a new OpenZiti identity. The previous identity is cleaned up by Ziti Management lease GC.

**Agent identities** are ephemeral — created by the Orchestrator via Ziti Management before each pod starts, and deleted when the pod stops. See [Agent Identity Lifecycle](#agent-identity-lifecycle) below.

### Agent Identity Lifecycle

Agent pods are short-lived. Their OpenZiti identities are created and destroyed with the pod.

1. The Agents Orchestrator creates an OpenZiti identity via the Ziti Management service before requesting the pod.
2. The Orchestrator passes the enrollment JWT to Runner as part of `StartWorkload` configuration.
3. Runner starts the pod with the JWT. The Ziti sidecar enrolls on startup, receiving an x509 certificate.
4. All API calls from processes in the pod reach platform APIs via OpenZiti service hostnames resolved by the sidecar DNS server and intercepted via TPROXY.
5. When the Orchestrator stops the workload, it deletes the OpenZiti identity via Ziti Management. The certificate becomes invalid.

The Runner treats the enrollment JWT as opaque configuration. See [OpenZiti Integration](openziti.md) for the full lifecycle diagram.

### OpenZiti Identities

| Identity | Lifecycle | Provisioning | Calls via OpenZiti |
|----------|-----------|-------------|--------------------|
| Agents Orchestrator | Ephemeral (per pod) | Self-enrollment via Ziti Management | Runner |
| Runner | Persistent (enrolled via service token) | Service token (Terraform/CLI → enrollment endpoint) | — (binds service, receives work) |
| Agent pod (Ziti sidecar) | Ephemeral (per pod) | Orchestrator via Ziti Management | Gateway, LLM Proxy, Tracing |
| App | Persistent (enrolled via service token) | Service token (Terraform/CLI → enrollment endpoint) | Gateway (dial) + own service (bind) |
| Gateway | Ephemeral (per pod) | Self-enrollment via Ziti Management | — (binds service, receives connections) |
| LLM Proxy | Ephemeral (per pod) | Self-enrollment via Ziti Management | — (binds service, receives connections) |
| Tracing | Ephemeral (per pod) | Self-enrollment via Ziti Management | — (binds service, receives connections) |
| Ziti Management | N/A — no OpenZiti network identity | Controller API credential (Terraform) | OpenZiti Controller (via Istio, not overlay) |

## Two Network Layers

The platform uses two network layers.

```mermaid
graph TB
    subgraph OpenZiti Overlay
        direction LR
        Runner[Runner]
        AgentExt[Agent Pod + Ziti Sidecar<br/>external]
        App[App]
    end

    subgraph Kubernetes Cluster
        subgraph Istio Mesh
            GW[Gateway]
            Chat[Chat]
            Threads[Threads]
            Files[Files]
            Notif[Notifications]
            LLMProxy[LLM Proxy]
            Other[Other Services]
        end

        subgraph "Istio + OpenZiti SDK"
            Orch[Agents Orchestrator]
        end

        AgentInt[Agent Pod + Ziti Sidecar<br/>internal]
        ZR[OpenZiti Router]
    end

    Runner -.->|OpenZiti| ZR
    AgentExt -.->|OpenZiti| ZR
    App -.->|OpenZiti| ZR
    AgentInt -.->|OpenZiti| ZR
    Orch -.->|"OpenZiti (SDK)"| Runner

    ZR -->|Istio| GW
    ZR -->|Istio| LLMProxy
    GW -->|Istio| Chat & Threads & Files & Notif & Other
    LLMProxy -->|Istio| Other
    Orch -->|Istio| Threads & Other
```

### SDK Embedding

**Infrastructure services** that participate in both the Istio mesh and the OpenZiti overlay use the **OpenZiti Go SDK** embedded in the application process — not an OpenZiti sidecar or tunneler. This avoids conflicts between the Istio sidecar proxy and an OpenZiti sidecar competing for outbound traffic routing.

Agent pods are not in the Istio mesh. They use a Ziti sidecar container within the pod to enroll the agent identity, resolve OpenZiti service hostnames via DNS, and transparently intercept traffic with TPROXY.

| Service | OpenZiti SDK Usage | Istio |
|---------|-------------------|-------|
| **Agents Orchestrator** | Dials runners via `zitiContext.Dial("runner-{runnerId}")` | All other outbound calls (Threads, Agents, Secrets, etc.) |
| **Gateway** | Binds `gateway` service via `zitiContext.ListenWithOptions("gateway", ...)` | All outbound calls to internal services |
| **LLM Proxy** | Binds `llm-proxy` service via `zitiContext.ListenWithOptions("llm-proxy", ...)` | All outbound calls to internal services (LLM, Users, Authorization, Ziti Management) |
| **Tracing** | Binds `tracing` service via `zitiContext.ListenWithOptions("tracing", ...)` | All outbound calls to internal services (Agents, Authorization, Ziti Management) |

The OpenZiti Go SDK implements Go's standard `net.Listener` and `net.Conn` interfaces. A gRPC server can accept connections from an OpenZiti listener the same way it accepts connections from a TCP listener. Similarly, a gRPC client can dial through an OpenZiti context the same way it dials a TCP address. See [`openziti/sdk-golang`](https://github.com/openziti/sdk-golang).

### Istio — Internal Service Mesh

Istio provides mTLS between all pods within the Kubernetes cluster. Identity is based on Kubernetes ServiceAccounts.

| Concern | Mechanism |
|---------|-----------|
| Pod-to-pod mTLS | Automatic via sidecar/ambient mode |
| Identity model | SPIFFE certificates from ServiceAccounts |
| Policy enforcement | `PeerAuthentication` (strict mTLS), `AuthorizationPolicy` (service-level access) |
| Scope | Within the cluster only |

### OpenZiti — Cross-Boundary Overlay

OpenZiti provides identity and connectivity for actors outside the cluster or needing application-level identity.

| Concern | Mechanism |
|---------|-----------|
| mTLS | Per-identity x509 certificates from OpenZiti Controller |
| Identity model | Platform-managed identities (agent ID, runner ID, app ID) |
| Policy enforcement | OpenZiti service policies (which identity can dial which service) |
| Scope | Cross-boundary (runners, agents, apps) and internal (agents in cluster, orchestrator-to-runner) |

### Why Both

**Istio** secures internal service-to-service communication. It knows nothing about application-level identity (which specific agent, which organization).

**OpenZiti** provides application-level identity and connectivity for actors that cross the cluster boundary (runners, agents, apps) and for connections that must use a uniform protocol regardless of location (orchestrator-to-runner).

They operate on different connections:

| Connection | Layer | Notes |
|------------|-------|-------|
| Agent → Gateway | OpenZiti | Agents always connect via overlay, regardless of location (via Ziti sidecar) |
| Agent → LLM Proxy | OpenZiti | LLM API calls via overlay, regardless of location (via Ziti sidecar) |
| Agent → Tracing | OpenZiti | Span export via overlay, regardless of location (via Ziti sidecar) |
| App → Gateway | OpenZiti | Apps always connect via overlay |
| Orchestrator → Runner | OpenZiti (SDK) | Uniform protocol for all runners |
| Orchestrator → Threads, Agents, etc. | Istio | Standard internal service calls |
| Gateway → internal services | Istio | Standard internal service calls |
| LLM Proxy → internal services | Istio | LLM service, Users, Authorization, Ziti Management |
| Tracing → internal services | Istio | Agents service, Authorization, Ziti Management |
| Internal service → internal service | Istio | Standard internal service calls |

## Authentication Boundary

**External traffic**: Authenticated at the **Gateway**, the **[LLM Proxy](llm-proxy.md)**, or the **[Tracing](tracing.md)** service. Users via OIDC access token validation (JWT signature verified against IdP JWKS, identity resolved through [Users](users.md)) or via [API token](api-tokens.md) (opaque token, identity resolved through [Users](users.md)). Agents, Runners, Apps via OpenZiti mTLS (identity extracted via [Ziti Management](openziti.md)). The LLM Proxy authenticates agents via OpenZiti mTLS or API token for LLM API calls. The Tracing service authenticates agents via OpenZiti mTLS for span ingestion. Organization membership is not validated at the authentication boundary — it is enforced by the [authorization model](authz.md) at the service level.

**Internal traffic**: Authenticated by **Istio** mTLS (service identity from ServiceAccount). End-user/agent identity propagated in gRPC metadata after Gateway authentication.

## Participants and Identities

[Threads](threads.md) identifies participants by opaque `identity_id` UUIDs — it operates on IDs only. [Chat](chat.md) resolves identity types via the [Identity](identity.md) service, then fetches profiles from [Users](users.md) (for users) or [Agents](agents-service.md) (for agents).

## CLI Authentication

All platform CLI tools ([`agyn`](agyn-cli.md), [`agynd`](agynd-cli.md), [`agn`](agn-cli.md)) use the same authentication convention with two methods and a fixed priority order. The auth token stored in `~/.agyn/credentials` is any token the Gateway accepts — an OIDC access token from a login flow or a long-lived [API token](api-tokens.md) created via `agyn auth create-token`.

| Priority | Method | Mechanism | When Used |
|----------|--------|-----------|-----------|
| 1 | **Network identity (Ziti sidecar)** | Pod-level [OpenZiti](#network-identity-openziti) mTLS via the Ziti sidecar | Automatic when the environment provides an enrolled OpenZiti identity (e.g., inside agent pods with a Ziti sidecar) |
| 2 | **Auth token** | Token stored in `~/.agyn/credentials`, sent to the [Gateway](gateway.md) as a bearer token | Developer machines, CI, or any environment without OpenZiti |

### Resolution Order

1. If an OpenZiti identity is available in the environment, use it. All API calls go over the OpenZiti overlay with mTLS.
2. Otherwise, read the auth token from `~/.agyn/credentials` and attach it to Gateway requests.

### Token Storage

Tokens are stored in the user's home directory:

```
~/.agyn/credentials
```

The file contains the auth token used to authenticate against the Gateway. It is created by a login flow (e.g., `agyn auth login`) and read by all CLI tools.
