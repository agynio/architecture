# Resource Definitions

Canonical schema for all team-managed resources in the Agyn platform. This is the single source of truth for resource structure — the Terraform provider, Teams API, and UI should all align to these definitions.

Resources are managed by the Teams service and stored in PostgreSQL. Each resource has a common envelope (`id`, `title`, `description`) plus a resource-specific `config` object.

---

## Agent

An agent definition that determines how an agent workload behaves when processing thread messages.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | string | | Friendly name (max 64 chars) |
| `role` | string | | Role label (max 64 chars) |
| `model` | string | `"gpt-5"` | LLM model identifier |
| `systemPrompt` | string | `"You are a helpful AI assistant."` | System prompt injected at start of each turn |
| `debounceMs` | integer | `0` | Debounce window (ms) for message buffer. `0` = no debounce |
| `whenBusy` | enum | `"wait"` | `"wait"` — queue new messages for next turn. `"injectAfterTools"` — inject into current turn after tool calls |
| `processBuffer` | enum | `"allTogether"` | `"allTogether"` — process all queued messages at once. `"oneByOne"` — one message per turn |
| `sendFinalResponseToThread` | boolean | `true` | Auto-send final assistant response to the thread |
| `restrictOutput` | boolean | `false` | Enforce calling a tool before finishing the turn |
| `restrictionMessage` | string | `"Do not produce a final answer directly..."` | Instruction injected when `restrictOutput` is true |
| `restrictionMaxInjections` | integer | `0` | Max enforcement injections per turn. `0` = unlimited |
| `summarizationKeepTokens` | integer | `0` | Number of most-recent tokens to keep verbatim during summarization |
| `summarizationMaxTokens` | integer | `512` | Maximum token budget for generated summaries |

All fields are optional (the schema uses `.partial()`).

---

## MCP Server

An MCP server definition that describes how to start and connect to an MCP tool server inside a workspace container.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `namespace` | string | `""` | Namespace prefix for exposed tools. Tools are named `<namespace>_<toolName>` |
| `command` | string | `"mcp start --stdio"` | Startup command executed inside the container |
| `workdir` | string | | Working directory inside the container |
| `env` | array | | Environment variables: `[{name: string, value: string \| reference}]` |
| `requestTimeoutMs` | integer | | Per-request timeout (ms) |
| `startupTimeoutMs` | integer | `15000` | Startup handshake timeout (ms) |
| `heartbeatIntervalMs` | integer | | Interval for heartbeat pings (ms) |
| `staleTimeoutMs` | integer | | Staleness timeout for cached tools (ms). `0` = never stale |
| `restart.maxAttempts` | integer | `5` | Maximum restart attempts during resilient start |
| `restart.backoffMs` | integer | `2000` | Base backoff (ms) between restart attempts |

All fields are optional.

---

## Workspace Configuration

A workspace container configuration that defines the execution environment for agents and MCP servers.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `image` | string | | Container image override |
| `env` | array | | Environment variables: `[{name: string, value: string}]` |
| `initialScript` | string | | Shell script (`/bin/sh -lc`) to run after container creation |
| `cpu_limit` | string \| number | | CPU limit (cores as number or millicores as string, e.g., `"500m"`) |
| `memory_limit` | string \| number | | Memory limit (bytes as number or with units: `Ki`, `Mi`, `Gi`, `Ti`, `KB`, `MB`, `GB`, `B`) |
| `platform` | enum | | Docker platform: `"linux/amd64"`, `"linux/arm64"`, `"auto"` |
| `enableDinD` | boolean | `false` | Enable per-workspace Docker-in-Docker sidecar |
| `ttlSeconds` | integer | `86400` | Idle TTL (seconds) before workspace cleanup. `0` or negative = no cleanup |
| `nix` | object | | Nix metadata (opaque — managed by UI) |
| `volumes.enabled` | boolean | `false` | Enable persistent named volume mount |
| `volumes.mountPath` | string | `"/workspace"` | Absolute container path for the volume mount |

All fields are optional.

---

## Memory Bucket

A memory scope definition for persistent agent memory (vector/KV storage).

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `scope` | enum | `"global"` | `"global"` — shared across all threads. `"perThread"` — isolated per thread |
| `collectionPrefix` | string | | Optional prefix for the underlying collection name |

---

## Attachment

A typed relationship between two resources. Attachments connect MCP servers, workspaces, tools, and memory buckets to agents.

| Field | Type | Description |
|-------|------|-------------|
| `kind` | string | Relationship type |
| `source_id` | string (UUID) | Source resource |
| `target_id` | string (UUID) | Target resource |

Attachment has no `config` — the relationship is fully described by the kind and the two resource IDs.

### Attachment Kinds

| Kind | Source → Target | Description |
|------|----------------|-------------|
| `tool_agent` | MCP Server / Tool / Workspace → Agent | Connects a tool provider to an agent |

---

## Environment Variable References

Several resources accept environment variables. The `value` field supports two forms:

**Plain string:**
```json
{ "name": "API_KEY", "value": "sk-..." }
```

**Vault reference:**
```json
{
  "name": "API_KEY",
  "value": {
    "kind": "vault",
    "mount": "secret",
    "path": "platform/keys",
    "key": "api_key"
  }
}
```

**Variable reference:**
```json
{
  "name": "API_KEY",
  "value": {
    "kind": "variable",
    "key": "api_key"
  }
}
```

Vault and variable references are resolved at runtime by the platform. The resolved value is never stored in config or state.
