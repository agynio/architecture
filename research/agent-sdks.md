# Agent SDKs Research

Research on programmatic SDKs for Codex and Claude Code — the two third-party agent CLIs that `agynd` needs to control. Includes protocol analysis, SDK sizing, and the architecture decision for Go integration.

## Overview

Both Codex and Claude Code provide SDKs in **TypeScript and Python**. **Neither has an official Go SDK.** TypeScript SDKs are used as the baseline for comparison since they are the most mature.

However, the two agents use **different wire protocols**:

- **Codex** has a documented **JSON-RPC v2** protocol (`codex app-server`) with a machine-readable JSON Schema.
- **Claude Code** uses a **custom JSONL** protocol (no formal spec; defined implicitly by SDK source code).

| Aspect | Codex | Claude Code |
|--------|-------|-------------|
| Package (TS) | `@openai/codex-sdk` | `@anthropic-ai/claude-agent-sdk` |
| Package (Python) | `codex-app-server-sdk` (experimental) | `claude-agent-sdk` (stable) |
| Official Go SDK | None | None |
| Community Go SDKs | None found | `severity1/claude-code-sdk-go` (full parity), `jrossi/claude-code-sdk-golang` |
| Protocol | **JSON-RPC v2** over subprocess stdio | **Custom JSONL** over subprocess stdio |
| Protocol docs | ✅ Documented at [developers.openai.com/codex/app-server](https://developers.openai.com/codex/app-server/) + JSON Schema in repo | ❌ No spec; reverse-engineer from SDK types |
| License | Apache-2.0 | Commercial (Anthropic ToS) |

## Protocol Details

### Codex: JSON-RPC v2 (via `codex app-server`)

Codex exposes **two** programmatic interfaces:

1. **`codex exec --json`** — simple one-shot JSONL stream. Events: `thread.started`, `turn.started`, `turn.completed`, `turn.failed`, `item.*`, `error`. This is what the TS SDK wraps.
2. **`codex app-server`** — full bidirectional JSON-RPC v2 protocol. Requests have `method`/`params`/`id`, responses echo `id` with `result` or `error`, notifications omit `id`. This is what the Python SDK uses.

The app-server protocol is the proper programmatic API:

- **Documented**: [developers.openai.com/codex/app-server](https://developers.openai.com/codex/app-server/)
- **JSON Schema in repo**: `codex-rs/app-server-protocol/schema/json/codex_app_server_protocol.v2.schemas.json` (12,141 lines)
- **Schema auto-generation**: `codex app-server generate-json-schema --out ./schemas`
- **~530 generated types** (from schema), **~50 notification types**, **~10 request methods**
- **Transports**: stdio (default), WebSocket (experimental)

Request example:
```json
{"method": "thread/start", "id": 10, "params": {"model": "gpt-5.1-codex"}}
```

Response:
```json
{"id": 10, "result": {"thread": {"id": "thr_123"}}}
```

Notification:
```json
{"method": "turn/completed", "params": {"turn": {"id": "turn_456"}}}
```

Key request methods: `initialize`, `thread/start`, `thread/list`, `thread/resume`, `thread/fork`, `thread/archive`, `turn/start`, `turn/interrupt`, `models`.

Key notifications: `item/agentMessage/delta`, `item/completed`, `item/started`, `turn/completed`, `turn/started`, `thread/started`, `hook/started`, `hook/completed`, `error`, plus ~40 others.

### Claude Code: Custom JSONL (via `claude` CLI)

Claude Code uses `--output-format stream-json --input-format stream-json --verbose` flags. The protocol is **not JSON-RPC** — there are no `method`/`id` fields, no request-response correlation.

**Inbound messages (stdout)** are discriminated by a `type` field:

| Type | Description |
|------|-------------|
| `assistant` | Assistant response with content blocks (text, thinking, tool_use, tool_result) |
| `user` | User message echo |
| `system` | System events — subtypes: `init`, `task_started`, `task_progress`, `task_notification` |
| `result` | Final result — subtypes: `success`, `error_during_execution`, `error_max_turns`, `error_max_budget_usd` |
| `stream_event` | Raw API streaming events (when `includePartialMessages` enabled) |
| `rate_limit_event` | Rate limit status |

**Outbound messages (stdin)** include initialize requests, user messages, and ~11 control request types (`interrupt`, `permission`, `set_permission_mode`, `hook_callback`, `mcp_message`, `rewind_files`, `mcp_reconnect`, `mcp_toggle`, `stop_task`, etc.).

**No formal spec exists.** The protocol is defined by:
- Python SDK `types.py`: 1,203 lines, 82 type classes
- Python SDK `message_parser.py`: 251 lines (the actual parsing logic)
- Python SDK `subprocess_cli.py`: 631 lines (transport layer)
- TS SDK reference: [platform.claude.com/docs/en/agent-sdk/typescript](https://platform.claude.com/docs/en/agent-sdk/typescript) (~20 message types in `SDKMessage` union)

## SDK Size Analysis

### Codex

| Component | Lines | Files |
|-----------|-------|-------|
| TS SDK (total source) | 918 | 10 |
| Python SDK (total source) | 8,283 | 10 |
| — generated protocol types (`v2_all.py`) | 6,351 | 1 |
| — actual logic | ~1,900 | 9 |
| JSON Schema (v2, in repo) | 12,141 | 1 |

Largest TS files: `exec.ts` (389), `thread.ts` (155), `items.ts` (127), `events.ts` (80).

### Claude Code

| Component | Lines | Files |
|-----------|-------|-------|
| Python SDK (total source) | 5,331 | 15 |
| — types | 1,203 | 1 |
| — transport (subprocess_cli.py) | 631 | 1 |
| — message parser | 251 | 1 |
| — session/control logic | ~2,100 | 3 |
| — client + query (public API) | ~620 | 2 |

### Community Go SDKs (Claude Code only)

| Repo | Non-test Go lines | Files | Coverage |
|------|-------------------|-------|----------|
| `severity1/claude-code-sdk-go` | 6,805 | ~30 | Full Python SDK parity (permissions, hooks, MCP, sessions) |
| `jrossi/claude-code-sdk-golang` | 1,998 | ~12 | Core functionality |

### Go Implementation Effort Estimate

| | Codex | Claude Code |
|---|---|---|
| **Types** | Auto-generate from JSON Schema (~530 types) | Hand-write from Python/TS (~82 types) |
| **Transport** | ~500 lines Go (JSON-RPC v2 + subprocess) | ~500 lines Go (custom JSONL + subprocess) |
| **Full SDK estimate** | ~1,500 lines Go | ~3,000–6,800 lines Go |
| **Effort (from scratch)** | ~1 week | ~2–3 weeks |
| **Existing assets** | JSON Schema in Codex repo | Community Go SDKs to evaluate/fork |

## SDK Capabilities Comparison

### Core Lifecycle

| Capability | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **Create client** | `new Codex(options?)` | — (no client object; uses `query()` function) |
| **Start session** | `codex.startThread(options?)` → `Thread` | `query({prompt, options})` → `Query` (async generator) |
| **Send prompt (buffered)** | `thread.run(prompt, options?)` → `Turn` | Iterate `query()` generator to completion → `SDKResultMessage` |
| **Send prompt (streaming)** | `thread.runStreamed(prompt, options?)` → `{events: AsyncGenerator}` | Same `query()` — it is always an async generator of `SDKMessage` |
| **Resume session** | `codex.resumeThread(threadId)` → `Thread` | `options.resume = sessionId` on next `query()` call |
| **Cancel / abort** | `AbortSignal` via `options.signal` | `AbortController` via `options.abortController`; also `query.interrupt()` |
| **Multi-turn conversation** | Call `thread.run()` repeatedly on same `Thread` | Streaming input mode: yield `SDKUserMessage` from an `AsyncGenerator` passed as prompt |

### Session / Thread Management

| Capability | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **Session persistence** | Automatic (`~/.codex/sessions`); resume by thread ID | Automatic (`~/.claude/projects/`); resume by session ID |
| **V2 Session API** | — (single API) | `unstable_v2_createSession(options)` → `SDKSession` |
| **V2 resume** | — | `unstable_v2_resumeSession(sessionId, options)` |
| **V2 one-shot** | — | `unstable_v2_prompt(message, options)` → `SDKResultMessage` |
| **Session send/receive** | — | `session.send(message)` / `session.receive()` → async generator |
| **Session close** | — (thread is GC'd) | `session.close()` or `Symbol.asyncDispose` |
| **Fork session** | — | `options.forkSession = true` |

### Configuration

| Capability | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **Working directory** | `startThread({workingDirectory})` | `options.cwd` |
| **Model selection** | Via CLI config / environment | `options.model`, `options.fallbackModel` |
| **System prompt** | Via CLI config | `options.systemPrompt` (string or preset + append) |
| **Environment variables** | `new Codex({env})` | `options.env` |
| **Config overrides** | `new Codex({config})` — flattened TOML keys | `options.extraArgs` — arbitrary CLI args |
| **Max turns** | Via CLI config | `options.maxTurns` |
| **Max budget** | Via CLI config | `options.maxBudgetUsd` |
| **Max thinking tokens** | Via CLI config | `options.maxThinkingTokens` |
| **Skip git repo check** | `startThread({skipGitRepoCheck: true})` | — |
| **Additional directories** | — | `options.additionalDirectories` |
| **Settings sources** | — | `options.settingSources` (user/project/local) |

### Output / Results

| Capability | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **Final text response** | `turn.finalResponse` | `resultMessage.result` (when `subtype === 'success'`) |
| **Structured output (JSON schema)** | `thread.run(prompt, {outputSchema})` | `options.outputFormat = {type: 'json_schema', schema}` |
| **Structured output result** | `turn.finalResponse` (parsed) | `resultMessage.structured_output` |
| **Turn items / events** | `turn.items` | Stream of `SDKMessage` (assistant, tool_progress, system, etc.) |
| **Usage / cost tracking** | `event.usage` (in streaming) | `resultMessage.total_cost_usd`, `resultMessage.usage`, `resultMessage.modelUsage` |
| **Duration tracking** | — | `resultMessage.duration_ms`, `resultMessage.duration_api_ms` |
| **Error types** | Thrown exceptions | `resultMessage.subtype`: `error_during_execution`, `error_max_turns`, `error_max_budget_usd` |
| **Image input** | `thread.run([{type: "local_image", path}])` | — (via tool use / prompt content) |

### Tool and Permission Control

| Capability | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **Tool allow-list** | — (controlled via CLI config) | `options.tools` or `options.allowedTools` |
| **Tool deny-list** | — | `options.disallowedTools` |
| **All built-in tools preset** | — | `{type: 'preset', preset: 'claude_code'}` |
| **Permission modes** | — (CLI `--approval` flag) | `options.permissionMode`: `default`, `acceptEdits`, `bypassPermissions`, `plan`, `dontAsk` |
| **Custom permission handler** | — | `options.canUseTool(toolName, input, opts)` → allow/deny |
| **Permission via MCP tool** | — | `options.permissionPromptToolName` |
| **Sandbox settings** | Via CLI config | `options.sandbox = {enabled, autoAllowBashIfSandboxed, network, ...}` |

### Built-in Tools

| Tool Category | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **File read** | Built-in (via CLI) | `Read` — `{file_path, offset?, limit?}` |
| **File write** | Built-in (via CLI) | `Write` — `{file_path, content}` |
| **File edit** | Built-in (via CLI, apply-patch) | `Edit` — `{file_path, old_string, new_string}` |
| **Shell / bash** | Built-in (via CLI) | `Bash` — `{command, timeout?, description?}` |
| **File search (glob)** | Built-in (via CLI) | `Glob` — `{pattern, path?}` |
| **Content search (grep)** | Built-in (via CLI) | `Grep` — `{pattern, path?, glob?}` |
| **Web search** | Built-in (via CLI) | `WebSearch` — `{query, allowed_domains?}` |
| **Web fetch** | Built-in (via CLI) | `WebFetch` — `{url, prompt}` |
| **Task management** | — | `TodoWrite` — `{todos: [{content, status}]}` |
| **Notebook edit** | — | `NotebookEdit` — `{notebook_path, new_source, ...}` |
| **Sub-agent invocation** | — | `Agent` — `{description, prompt, subagent_type, model?}` |
| **Ask user question** | — | `AskUserQuestion` — `{questions: [...]}` |
| **Background shell** | — | `BashOutput` / `KillShell` — read/terminate background processes |

### MCP Integration

| Capability | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **MCP server config** | Via CLI config file | `options.mcpServers` — inline stdio or HTTP configs |
| **Strict MCP config** | — | `options.strictMcpConfig` |
| **List MCP resources** | — | `ListMcpResources` tool |
| **Read MCP resource** | — | `ReadMcpResource` tool |
| **MCP server status** | — | `query.mcpServerStatus()` |

### Hooks and Extensibility

| Capability | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **Hook system** | — | `options.hooks` — callbacks for lifecycle events |
| **Pre-tool-use hook** | — | `PreToolUse` — inspect/modify/block before tool runs |
| **Post-tool-use hook** | — | `PostToolUse` / `PostToolUseFailure` |
| **Session lifecycle hooks** | — | `SessionStart`, `SessionEnd`, `Stop` |
| **User prompt hook** | — | `UserPromptSubmit` |
| **Sub-agent hooks** | — | `SubagentStart`, `SubagentStop` |
| **Compaction hook** | — | `PreCompact` |
| **Custom agents (sub-agents)** | — | `options.agents` — define named agents with tools/prompts/models |
| **Plugins** | — | `options.plugins` — local plugin configs |

### Runtime Query Methods

| Capability | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **Interrupt execution** | Abort via signal | `query.interrupt()` |
| **Change permission mode** | — | `query.setPermissionMode(mode)` |
| **Switch model** | — | `query.setModel(model)` |
| **Set thinking tokens** | — | `query.setMaxThinkingTokens(n)` |
| **List available commands** | — | `query.supportedCommands()` |
| **List available models** | — | `query.supportedModels()` |
| **Get account info** | — | `query.accountInfo()` |
| **Stream additional input** | — | `query.streamInput(asyncIterable)` |

### Message Types (Streaming)

| Message Type | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **Item completed** | `event.type === "item.completed"` | `msg.type === 'assistant'` |
| **Turn completed** | `event.type === "turn.completed"` | `msg.type === 'result'` |
| **System / init** | — | `msg.type === 'system'` (subtype: `init`) |
| **Tool progress** | — (via items) | `msg.type === 'tool_progress'` |
| **Auth status** | — | `msg.type === 'auth_status'` |
| **Partial / streaming** | — (via `runStreamed` events) | `msg.type === 'partial_assistant'` (if `includePartialMessages`) |
| **Compact boundary** | — | `msg.type === 'compact_boundary'` |
| **Hook response** | — | `msg.type === 'hook_response'` |

## Key Differences Summary

1. **API shape**: Codex uses an object-oriented model (`Codex` → `Thread` → `run()`), while Claude Code uses a functional model (`query()` → async generator).

2. **Feature surface**: Claude Code Agent SDK exposes significantly more control — custom permissions, hooks, sub-agents, tool allow/deny lists, sandbox configuration, budget limits, MCP server management, and runtime query methods. Codex SDK is deliberately minimal: create thread, run prompt, get result.

3. **Streaming model**: Codex offers two modes (`run()` for buffered, `runStreamed()` for streaming). Claude Code always streams via async generator with a richer set of message types.

4. **Configuration**: Codex delegates most configuration to CLI config files and environment variables. Claude Code exposes nearly everything as SDK options.

5. **Protocol**: Codex uses JSON-RPC v2 (documented, schema-driven). Claude Code uses a custom JSONL protocol (undocumented, reverse-engineered from SDK source).

## References

- Codex app-server protocol docs: https://developers.openai.com/codex/app-server/
- Codex app-server JSON Schema: https://github.com/openai/codex/blob/main/codex-rs/app-server-protocol/schema/json/codex_app_server_protocol.v2.schemas.json
- Codex non-interactive mode docs: https://developers.openai.com/codex/noninteractive/
- Codex SDK TypeScript source: https://github.com/openai/codex/tree/main/sdk/typescript
- Codex SDK Python source: https://github.com/openai/codex/tree/main/sdk/python
- Codex SDK docs: https://developers.openai.com/codex/sdk/
- Claude Code CLI reference: https://code.claude.com/docs/en/cli-reference
- Claude Code agent loop docs: https://platform.claude.com/docs/en/agent-sdk/agent-loop
- Claude Code Agent SDK TypeScript on npm: https://www.npmjs.com/package/@anthropic-ai/claude-agent-sdk
- Claude Code Agent SDK Python on PyPI: https://pypi.org/project/claude-agent-sdk/
- Claude Code Agent SDK Python source: https://github.com/anthropics/claude-agent-sdk-python
- Claude Code Agent SDK docs: https://platform.claude.com/docs/en/agent-sdk/overview
- Claude Code Agent SDK TS reference (message types): https://platform.claude.com/docs/en/agent-sdk/typescript
- Claude Code Agent SDK Python reference: https://platform.claude.com/docs/en/agent-sdk/python
- Community Go SDK (severity1): https://github.com/severity1/claude-code-sdk-go
- Community Go SDK (jrossi): https://github.com/jrossi/claude-code-sdk-golang

## Architecture Decision: Agent SDK Strategy

### Context

`agynd` is a Go daemon that orchestrates multiple agent implementations (Codex, Claude Code, agn). Each agent runs as a subprocess. agynd needs Go SDKs to communicate with each agent's CLI.

### Decision

Three separate Go SDK modules, all consumed by agynd as dependencies:

```
agynd
├── imports codex-sdk-go     → spawns `codex app-server`  → JSON-RPC v2 over stdio
├── imports claude-sdk-go    → spawns `claude`             → custom JSONL over stdio
└── imports agn-sdk-go       → spawns `agn`                → JSON-RPC v2 over stdio
```

1. **`codex-sdk-go`** — standalone Go module for Codex integration.
   - Auto-generate types from the Codex JSON Schema (12K lines, 530 types).
   - Implement JSON-RPC v2 transport over subprocess stdio.
   - Use `codex app-server` (not `codex exec --json`) as the programmatic interface.

2. **`claude-sdk-go`** — standalone Go module for Claude Code integration.
   - Reverse-engineer types from Python SDK `types.py` (82 classes) and TS SDK reference.
   - Implement custom JSONL transport over subprocess stdio.
   - Evaluate community Go SDKs (`severity1/claude-code-sdk-go`) as starting point or reference.

3. **`agn-sdk-go`** — Go SDK module exported from the agn repository.
   - agn defines its own JSON-RPC v2 schema.
   - The SDK spawns `agn` as a subprocess, same pattern as Codex SDK.
   - agynd imports the SDK module, not agn's internal logic.

### Rationale

- **Codex and agn share JSON-RPC v2**: standard protocol with request-response correlation, formal error handling, and schema-driven type generation. A shared JSON-RPC v2 transport library can serve both SDKs.
- **Claude Code uses a custom protocol**: no JSON-RPC, no formal spec. Requires a separate transport implementation. The protocol is reverse-engineered from SDK source.
- **Separate modules**: each SDK is independently versioned and reusable outside agynd. agynd has zero protocol logic — it only talks through SDK interfaces.
- **agn SDK lives in agn repo**: agn controls its own schema and SDK, ensuring they stay in sync with the CLI.

### Consequences

- Three Go modules to build and maintain.
- Codex SDK can be largely auto-generated; Claude Code SDK requires more manual work.
- Protocol changes in Codex are caught by re-generating from the schema. Protocol changes in Claude Code require monitoring SDK source for breaking changes.
- A shared JSON-RPC v2 transport package can reduce duplication between Codex SDK and agn SDK.
