# agynd-cli

## Overview

`agynd` is the agent wrapper daemon. It bridges any agent CLI with the platform by connecting to the [Gateway](gateway.md) and the [LLM Proxy](llm-proxy.md) via OpenZiti service hostnames (for example, `gateway.ziti`, `llm-proxy.ziti`) that are transparently intercepted by the pod's [Ziti sidecar](openziti.md#agent-access-scope), preparing the agent runtime environment, and managing the agent process lifecycle. The [Runner](runner.md) starts `agynd` as the main process in an agent container.

| Aspect | Details |
|--------|---------|
| Binary name | `agynd` |
| Repository | `agynio/agynd-cli` |
| Language | Go |
| Role | Agent container entrypoint — bridges agent CLI with platform services |

## Responsibilities

### 1. Platform Connection

`agynd` implements the [agent contract](agent/overview.md):

- Subscribes to `thread_participant:{agentId}` room via [Gateway](gateway.md) → [Notifications](notifications.md) (server-streaming).
- Pulls unacknowledged messages via `GetUnackedMessages` (Gateway → [Threads](threads.md)).
- Posts agent responses back to the thread via `SendMessage`.
- Acknowledges processed messages via `AckMessages`.
- Follows the [Consumer Sync Protocol](notifications.md#consumer-sync-protocol) for reliable message delivery.

### 2. Message Formatting

`agynd` translates thread messages into the format expected by the agent CLI before feeding them via the SDK. Thread messages contain structured data (`body`, `files[]`), but agent CLIs receive plain text.

When a thread message has file attachments, `agynd` appends `agynfile://` URIs after the message body. See [Media — Message Formatting for LLM](media.md#message-formatting-for-llm).

```
What's in this image?
agynfile://file-uuid-1
```

Messages without file attachments are sent as the `body` field only. The `agynfile://` scheme is only appended when the `files` array is non-empty.

The agent CLI has no knowledge of thread messages, file IDs, or the `files` array — it receives pre-formatted plain text.

### 3. Environment Preparation

Before spawning the agent CLI, `agynd` fetches the agent configuration from the platform and prepares the runtime environment. The preparation is agent-specific — different agent CLIs expect different configuration conventions:

| Preparation | Description |
|-------------|-------------|
| **Skills** | Loads [skill](resource-definitions.md#skill) content and places it into the filesystem in the directory structure expected by the agent CLI |
| **LLM endpoint** | Provides [LLM Proxy](llm-proxy.md) endpoint configuration so the agent CLI knows where to make model calls |
| **MCP tools** | Configures the agent CLI with [MCP](mcp.md) server endpoints (`localhost:<port>` per server) so the agent CLI connects to each MCP sidecar directly over streamable HTTP |
| **Tracing endpoint** | Runs a local [OTLP tracing proxy](tracing.md#agynd-tracing-proxy) on `localhost:4317` that injects `agyn.thread.id` and forwards spans to the [Tracing](tracing.md) service via `tracing.ziti` |

This approach mirrors how tools like Claude Code and Codex CLI receive their configuration — through filesystem conventions and environment rather than a custom protocol.

The configuration strategy per agent CLI (where skills are placed, how MCP servers are connected, what environment variables are set) is determined by the [Agent Init Container](agent-init.md) — the init image's `config.json` specifies which SDK module `agynd` uses.

### 4. Agent Process Management

`agynd` spawns the configured agent CLI as a child process and communicates with it through an SDK specific to each agent type.

The agent CLI can be:
- [`agn`](agn-cli.md) — our own agent loop implementation.
- Any 3rd-party CLI (Claude Code, Codex CLI, custom implementations).

## Agent Communication Protocol

`agynd` does not implement agent protocols directly. It imports a separate **Go SDK module** for each supported agent CLI. Each SDK handles subprocess spawning, protocol encoding/decoding, and message framing.

```
agynd
├── imports codex-sdk-go     → spawns `codex app-server`  → JSON-RPC v2 over stdio
├── imports claude-sdk-go    → spawns `claude`             → custom JSONL over stdio
└── imports agn-sdk-go       → spawns `agn serve`          → JSON-RPC v2 over stdio
```

### Protocol per agent

| Agent CLI | SDK Module | Protocol | Subprocess Command |
|-----------|-----------|----------|-------------------|
| **Codex** | `codex-sdk-go` | JSON-RPC v2 | `codex app-server` |
| **Claude Code** | `claude-sdk-go` | Custom JSONL | `claude --output-format stream-json --input-format stream-json --verbose` |
| **agn** | `agn-sdk-go` | JSON-RPC v2 | `agn serve` |

Only `codex-sdk-go` is integrated in the initial implementation; `claude-sdk-go` does not exist as a repo and `agn-sdk-go` lives in `agn-cli/sdk/` but is not wired into `agynd`, so Claude and agn return `unsupported` at runtime.

### SDK responsibilities

Each SDK module is responsible for:

- Spawning and managing the agent CLI subprocess.
- Encoding outbound messages (prompts, control requests) in the agent's wire format.
- Decoding inbound messages (responses, events, errors) from the agent's wire format.
- Exposing a Go API that `agynd` calls — `agynd` never touches raw protocol bytes.

### Protocol details

**Codex** uses a documented JSON-RPC v2 protocol via `codex app-server`. The protocol has a [machine-readable JSON Schema](https://github.com/openai/codex/blob/main/codex-rs/app-server-protocol/schema/json/codex_app_server_protocol.v2.schemas.json) (~530 types, ~50 notification types, ~10 request methods). Types for `codex-sdk-go` are auto-generated from this schema. Documentation: [developers.openai.com/codex/app-server](https://developers.openai.com/codex/app-server/).

**Claude Code** uses a custom JSONL protocol with no formal specification. The protocol is defined implicitly by the [Python SDK source](https://github.com/anthropics/claude-agent-sdk-python) and [TypeScript SDK reference](https://platform.claude.com/docs/en/agent-sdk/typescript). Types for `claude-sdk-go` are reverse-engineered from these sources. Inbound messages are discriminated by a `type` field (`assistant`, `user`, `system`, `result`, `stream_event`, `rate_limit_event`). Outbound messages include initialize requests, user messages, and control requests.

**agn** uses JSON-RPC v2, same protocol pattern as Codex. agn defines its own schema. The `agn-sdk-go` module is exported from the agn repository.

### Why separate SDK modules

- Each SDK is independently versioned and reusable outside `agynd`.
- `agynd` has zero protocol logic — all wire format details are encapsulated in the SDK.
- Codex and agn share JSON-RPC v2, enabling a shared transport library underneath both SDKs.
- Protocol changes in Codex are caught by re-generating from its JSON Schema. Protocol changes in Claude Code require monitoring SDK source.

## Authentication

`agynd` supports two authentication methods, with the same priority order used by all CLI tools in the platform (see [CLI Authentication](authn.md#cli-authentication)):

| Method | Mechanism | Use Case |
|--------|-----------|----------|
| **Network identity (Ziti sidecar)** | Pod-level [OpenZiti](authn.md#network-identity-openziti) mTLS via the Ziti sidecar — automatic when the sidecar is present | Primary. The Orchestrator creates an OpenZiti identity and passes the enrollment JWT via Runner. The Ziti sidecar enrolls on startup and transparently intercepts OpenZiti service hostnames via DNS + TPROXY |
| **Auth token** | Token stored in `~/.agyn/credentials` and sent to the [Gateway](gateway.md) | Development, testing, or environments without OpenZiti |

In production, the pod's Ziti sidecar handles OpenZiti enrollment and mTLS. `agynd` connects to Gateway and LLM Proxy using OpenZiti service hostnames (for example, `gateway.ziti`, `llm-proxy.ziti`); the sidecar resolves these names and transparently intercepts traffic via DNS + TPROXY, so `agynd` does not embed the OpenZiti SDK. The [agent identity lifecycle](authn.md#agent-identity-lifecycle) is managed by the Orchestrator. The enrollment JWT is consumed by the sidecar, not by `agynd`.

## Architecture

```mermaid
graph TB
    subgraph "Agent Pod"
        ZitiSidecar[Ziti Sidecar]
        subgraph "Agent Container"
            agynd[agynd]
            AgentCLI[Agent CLI<br/>agn / 3rd-party]
            TracingProxy[Tracing Proxy<br/>localhost:4317]
            Skills[Skills on filesystem]

            agynd -->|spawns via SDK| AgentCLI
            agynd --> TracingProxy
            Skills -->|read by| AgentCLI
            AgentCLI -->|OTLP spans| TracingProxy
        end
    end

    subgraph "MCP Sidecars"
        MCP1[MCP Server 1]
        MCP2[MCP Server 2]
    end

    subgraph Platform
        Gateway
        LLMProxy[LLM Proxy]
        Tracing[Tracing]
    end

    agynd -->|platform calls via OpenZiti hostname| Gateway
    AgentCLI -->|LLM calls via OpenZiti hostname| LLMProxy
    AgentCLI -->|streamable HTTP<br/>localhost:port| MCP1 & MCP2
    TracingProxy -->|enriched spans via OpenZiti hostname| Tracing
    ZitiSidecar -.->|OpenZiti mTLS| Gateway
    ZitiSidecar -.->|OpenZiti mTLS| LLMProxy
    ZitiSidecar -.->|OpenZiti mTLS| Tracing
```

## Lifecycle

```mermaid
sequenceDiagram
    participant R as Runner
    participant D as agynd
    participant GW as Gateway
    participant A as Agent CLI

    R->>D: Start pod (Ziti sidecar enrolls identity)
    Note over D: Ziti sidecar resolves OpenZiti hostnames and intercepts traffic (DNS + TPROXY)
    D->>D: Fetch agent configuration (via Gateway at gateway.ziti)
    D->>D: Prepare environment (skills, LLM Proxy config)
    D->>D: Configure MCP endpoints for agent CLI
    D->>GW: Subscribe to thread_participant:{agentId}
    D->>A: Spawn agent CLI via SDK

    loop Message processing
        D->>GW: GetUnackedMessages(agentId)
        GW-->>D: Messages
        D->>A: Feed messages via SDK
        A->>A: Process (LLM loop, tools, etc.)
        A-->>D: Events/responses via SDK
        D->>GW: SendMessage (post response)
        D->>GW: AckMessages
    end

    Note over D: Wait for notification or poll
    GW-->>D: message.created
    Note over D: Resume message processing
```
