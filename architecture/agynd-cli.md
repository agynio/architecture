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

`agynd` implements the [agent contract](agent/overview.md) on behalf of a specific [agent instance](agent-instances.md):

- Subscribes to `instance_inbox:me` via [Gateway](gateway.md) → [Notifications](notifications.md) (server-streaming). See [Self-Subscription Sentinel](notifications.md#self-subscription-sentinel) — Notifications rewrites `:me` to the caller's `identity_id` (this workload's instance id) before authorization, so `agynd` does not need to hard-code its id.
- Pulls unacknowledged [inbox items](agent-instances.md#inbox) via `GetUnackedInboxItems(instance_id: AGENT_INSTANCE_ID)` on the [Agents Service](agents-service.md) (via Gateway). Each item is tagged with `source_kind` (`thread` or `direct`), and — when routed from a thread — `thread_id`, `message_id`, and `sender_id`.
- Posts agent responses to threads via `Threads.SendMessage(thread_id, ...)`. Sends carry an explicit `thread_id`; an omitted one resolves server-side to the instance's [`default_thread_id`](agent-instances.md#default-thread). There is no "current thread" derived from the turn's inbox items — see [Agent Instances — Outbound](agent-instances.md#outbound).
- Posts the agent CLI's final turn text to the default thread when the class sets [`final_message = default_thread`](resource-definitions.md#agent). See [Final Turn Message](#6-final-turn-message).
- Acknowledges processed inbox items via `AckInboxItems(instance_id, item_ids)` — by default after the turn completes (responses posted). SDKs that cannot observe turn boundaries may ack once items are durably written to the agent's state volume. See [Agent Instances — Ack timing](agent-instances.md#inbox).
- Follows the [Consumer Sync Protocol](notifications.md#consumer-sync-protocol) for reliable delivery.
- Sends keepalive signals to the [Runners](runners.md) service (via [Gateway](gateway.md)) while the agent is actively processing. See [Activity Keepalive](#5-activity-keepalive).

### 2. Message Formatting

`agynd` translates inbox items into the format expected by the agent CLI before feeding them via the SDK. Inbox items contain structured data (`body`, `files[]`, `thread_id`, `sender_id`, `source_kind`), but agent CLIs receive plain text.

Every item is prefixed with a compact header identifying its source, so the LLM can address replies correctly:

```
thread: <thread-id-or-ref>
from: @sender-handle
What's in this image?
agyn://file/file-uuid-1
```

For `source_kind = direct` (app-written) items, `thread:` is replaced by `source: direct` — the item carries no thread. The agent either names a thread explicitly or falls back to the instance's [default thread](agent-instances.md#default-thread), which is also where the final turn message goes when the class enables it.

When an item has file attachments, `agynd` appends `agyn://file/` URIs after the message body. See [Media — Message Formatting for LLM](media.md#message-formatting-for-llm). Items without file attachments contain only the header + body. The `agyn://file/` scheme is only appended when the `files` array is non-empty.

The agent CLI has no knowledge of the inbox schema, file IDs, or the `files` array — it receives pre-formatted plain text with the thread-source header.

### 3. Environment Preparation

Before spawning the agent CLI, `agynd` fetches class configuration from the platform via the Gateway (`gateway.ziti`) using its own agent-instance OpenZiti identity. Authentication is handled at the network level by the pod's Ziti sidecar. `agynd` reads two identifiers from environment variables: `AGENT_INSTANCE_ID` (this workload's instance) and `AGENT_ID` (the class the instance was spawned from). Class configuration is fetched with `AGENT_ID`; inbox and workload calls use `AGENT_INSTANCE_ID`. The preparation is agent-specific — different agent CLIs expect different configuration conventions:

| Preparation | Description |
|-------------|-------------|
| **Skills** | Fetches skills via `ListSkills(agent_id)` and writes content to the filesystem in the directory structure expected by the agent CLI |
| **LLM endpoint** | Writes [LLM Proxy](llm-proxy.md) endpoint configuration into the agent CLI's config file so the agent CLI knows where to make model calls. See [LLM Endpoint Configuration](#llm-endpoint-configuration) |
| **MCP tools** | Configures the agent CLI with [MCP](mcp.md) server endpoints (`localhost:<port>` per server) from the `AGENT_MCP_SERVERS` env var so the agent CLI connects to each MCP sidecar directly over streamable HTTP |
| **Tracing endpoint** | Runs a local [OTLP tracing proxy](tracing.md#agynd-tracing-proxy) on `localhost:4317` that injects `agyn.agent_instance.id`, `agyn.workload.id`, and per-turn `agyn.thread.id` / `agyn.thread.message.id` (from the current turn's inbox items) and forwards spans to the [Tracing](tracing.md) service via `tracing.ziti` |
| **Init scripts** | Fetches [init scripts](resource-definitions.md#initscript) via `ListInitScripts(agent_id)` and executes each in creation order using the container's default shell. Each script runs with its working directory set to `WORKSPACE_DIR` when that variable is defined in the subprocess environment, and to `/tmp` otherwise. Runs after environment setup and before spawning the agent CLI. If a script exits with a non-zero code, the script name and stderr output are printed to the container's stderr and execution continues with the next script. |
| **PATH** | Prepends `/agyn-bin/cli` and `/agyn-bin` to `PATH` in the subprocess environment. `/agyn-bin/cli` makes the `agyn` platform CLI available by name; `/agyn-bin` makes `agynd` and the configured agent CLI binary (`codex`, `claude`, or `agn`) available by name to the agent process and any child commands it runs. |

All user-defined environment variables — both plain-text values and resolved secret values — are injected directly into the container by the [Agents Orchestrator](agents-orchestrator.md) at workload assembly time. `agynd` reads them from the process environment and does not call `ListENVs`.

This approach mirrors how tools like Claude Code and Codex CLI receive their configuration — through filesystem conventions and environment rather than a custom protocol.

The configuration strategy per agent CLI (where skills are placed, how MCP servers are connected, what environment variables are set) is determined by the [Agent Init Container](agent-init.md) — the init image's `config.json` specifies which SDK module `agynd` uses.

#### LLM Endpoint Configuration

`agynd` configures each agent CLI to use the [LLM Proxy](llm-proxy.md) as its LLM endpoint. The configuration method is agent-specific:

**Codex CLI** — `agynd` writes `~/.codex/config.toml` with a custom model provider pointing at the LLM Proxy, and sets `CODEX_HOME=~/.codex` and `OPENAI_API_KEY` in the subprocess environment:

```toml
model_provider = "platform"

[model_providers.platform]
name = "Agyn LLM"
base_url = "http://llm-proxy.ziti/v1"
env_key = "OPENAI_API_KEY"
wire_api = "responses"
```

The custom provider is used instead of overriding the built-in OpenAI provider via `OPENAI_BASE_URL` because the built-in provider triggers behaviors the LLM Proxy does not implement (remote compaction via `POST /responses/compact`, realtime WebSocket) and has `env_key: None` which prevents `OPENAI_API_KEY` from being used for Bearer authentication in the subprocess auth pipeline.

If `HOME` is empty in the Codex subprocess environment, `agynd` sets `HOME=/tmp` before spawning — Codex resolves `~/.codex` against `HOME` and needs a writable path. The platform does not inject `HOME` at the orchestrator level; image defaults and user-set [ENVs](resource-definitions.md#env) apply. This fallback is Codex-specific and does not apply to other SDKs.

**[`agn`](agn-cli.md)** — `agynd` writes `~/.agyn/agn/config.yaml` with the LLM Proxy endpoint:

```yaml
llm:
  endpoint: http://llm-proxy.ziti/v1
```

**Claude Code** — `agynd` writes `~/.claude/settings.json` with LLM Proxy configuration and full tool permissions:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://llm-proxy.ziti/v1",
    "ANTHROPIC_AUTH_TOKEN": "unused-ziti-mTLS",
    "DISABLE_AUTOUPDATER": "1",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  },
  "permissions": {
    "allow": [
      "Bash",
      "Read",
      "Write",
      "Edit",
      "Glob",
      "Grep",
      "WebFetch",
      "WebSearch",
      "Task",
      "TodoWrite",
      "NotebookEdit"
    ],
    "deny": []
  }
}
```

`ANTHROPIC_BASE_URL` overrides the API endpoint. `ANTHROPIC_AUTH_TOKEN` sets a custom `Authorization: Bearer` header value — when running with the Ziti sidecar, authentication is handled at the network level, so the value is unused but must be non-empty to suppress Claude Code's authentication prompt. `DISABLE_AUTOUPDATER` prevents background update checks. `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` disables telemetry and other non-essential outbound requests.

The `permissions` block grants all built-in tools without interactive confirmation. The platform provides isolation and security at the container level — the agent should be able to perform any filesystem, shell, and network action within its container.

Inside the platform, agents connect to the LLM Proxy using the `llm-proxy.ziti` OpenZiti hostname. The Ziti sidecar resolves the hostname and transparently intercepts connections via DNS + TPROXY, so agent CLI subprocesses connect with standard HTTP clients — no OpenZiti SDK required. When running with the Ziti sidecar, authentication is handled at the network level by the sidecar's mTLS — the `OPENAI_API_KEY` value is unused. Over the public endpoint (development, CI), the token must be a valid platform API token (`agyn_...`).

### 4. Agent Process Management

`agynd` spawns the configured agent CLI as a child process and communicates with it through an SDK specific to each agent type.

The agent CLI can be:
- [`agn`](agn-cli.md) — our own agent loop implementation.
- Any 3rd-party CLI (Claude Code, Codex CLI, custom implementations).

### 5. Activity Keepalive

`agynd` reports agent activity to the [Runners](runners.md) service for [idle timeout](runners.md#idle-timeout) enforcement. While the agent CLI is actively processing (executing LLM calls, running tools, producing output), `agynd` calls `TouchWorkload` on the [Runners](runners.md) service (via [Gateway](gateway.md)) every 10 seconds. This updates the `last_activity_at` timestamp on the workload record.

When the agent is idle (turn complete, waiting for new messages), `agynd` stops sending keepalives. The [Agents Orchestrator](agents-orchestrator.md) compares `last_activity_at` against the agent's [`idle_timeout`](resource-definitions.md#agent) and stops workloads that have been idle too long.

`agynd` determines activity state from the agent CLI SDK — it knows when the agent is processing a request vs. waiting for input. The keepalive is SDK-agnostic: regardless of which agent CLI is running (Codex, Claude Code, `agn`), `agynd` uses the same `TouchWorkload` mechanism.

### 6. Final Turn Message

An agent CLI ends a turn with plain assistant text. No agent CLI protocol carries a thread target on it — Codex, Claude Code, and `agn` all emit text — so this output is not addressable on its own. Anything the agent wants delivered to a named thread goes through an explicit send (a platform tool call or `agyn threads send`) during the turn.

When the class sets [`final_message`](resource-definitions.md#agent) to `default_thread`, `agynd` additionally posts that final text to the instance's [`default_thread_id`](agent-instances.md#default-thread), after the turn completes and before acking the turn's inbox items. The post is unconditional — `agynd` does not inspect whether the agent already sent something, which is why the default is `discard` for agents that manage their own sends. Nothing is posted when the final text is empty or the instance's default thread is NULL; `agynd` logs and continues to ack.

`agynd` never derives a target from the turn's inbox items. An instance handling a sub-thread reply is frequently answering a *different* thread than the one that woke it — see [Agent Instances — Outbound](agent-instances.md#outbound).

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
    D->>GW: GetAgent(agent_id) + ListSkills + ListInitScripts + ListMCPs
    GW-->>D: Class config, skills, init scripts, MCP definitions
    D->>D: Prepare environment (skills to filesystem, LLM Proxy config, MCP endpoints)
    D->>D: Execute init scripts in order (/bin/sh -lc each)
    D->>GW: Subscribe to instance_inbox:me
    D->>A: Spawn agent CLI via SDK

    loop Inbox processing
        D->>GW: GetUnackedInboxItems(instanceId)
        GW-->>D: Inbox items (tagged with thread/sender)
        D->>A: Feed items via SDK (with thread-source headers)
        A->>A: Process (LLM loop, tools, explicit sends)
        A-->>D: Final turn text via SDK (no thread target)
        opt final_message = default_thread
            D->>GW: Threads.SendMessage(default_thread_id, body, files)
        end
        D->>GW: AckInboxItems(instanceId, itemIds)
    end

    Note over D: Wait for notification or poll
    GW-->>D: message.created on instance_inbox:me
    Note over D: Resume inbox processing
```
