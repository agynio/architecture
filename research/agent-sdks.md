# Agent SDKs Research

Research on programmatic SDKs for Codex and Claude Code — the two 3rd-party agent CLIs that `agynd` needs to control.

## Overview

Both Codex and Claude Code provide SDKs in **TypeScript and Python**. **Neither has a Go SDK.** Both SDKs work by spawning their respective CLI as a child process and exchanging structured messages over stdin/stdout. This is important for our Go integration strategy.

| Aspect | Codex SDK | Claude Code Agent SDK |
|--------|-----------|----------------------|
| TypeScript package | `@openai/codex-sdk` | `@anthropic-ai/claude-agent-sdk` |
| Python package | `codex-app-server-sdk` (experimental) | `claude-agent-sdk` |
| Go SDK | None | None |
| Other languages | — | Rust crate (`claude-agents-sdk`, community) |
| TS mechanism | Spawns `codex` CLI, JSONL over stdin/stdout | Spawns `claude` CLI, JSONL over stdin/stdout |
| Python mechanism | Spawns `codex app-server`, JSON-RPC v2 over stdin/stdout | Spawns `claude` CLI, JSONL over stdin/stdout |
| Python maturity | Experimental (v0.2.0, `sdk/python` in monorepo) | Stable (`pip install claude-agent-sdk`, v0.2.x on PyPI) |
| License | Apache-2.0 | Commercial (Anthropic ToS) |

## SDK Language Details

### Codex

- **TypeScript** (`@openai/codex-sdk`): Stable. Wraps the `codex` CLI binary, exchanges JSONL events over stdin/stdout. Object-oriented API: `Codex` → `Thread` → `run()`.
- **Python** (`codex-app-server-sdk`): Experimental. Uses a different protocol — communicates with `codex app-server` over JSON-RPC v2 (not the CLI's JSONL). Provides `Codex` / `AsyncCodex` → `Thread` → `Turn`. Package name: `codex-app-server-sdk`; runtime dependency: `codex-cli-bin` (platform-specific binary). Python 3.10+.

### Claude Code

- **TypeScript** (`@anthropic-ai/claude-agent-sdk`): Stable. Spawns `claude` CLI, JSONL over stdin/stdout. Functional API: `query()` → async generator; plus experimental `unstable_v2_createSession()`.
- **Python** (`claude-agent-sdk`): Stable. Same CLI-wrapping mechanism. Two entry points: `query()` for stateless streaming, `ClaudeSDKClient` for interactive multi-turn sessions with custom tools and hooks. `pip install claude-agent-sdk`. Python 3.10+.

## Go Integration Strategy

Since neither SDK has Go support, we have two main options:

### Option 1: Port the CLI stdio protocol to Go

Both SDKs spawn CLI processes and exchange structured messages over stdio. We can replicate this in Go:

1. **Spawn the CLI binary** as a child process from Go (`os/exec`).
2. **Write structured commands** to the process stdin (JSONL for TS-style, or JSON-RPC v2 for Codex app-server).
3. **Read structured events** from the process stdout (line-delimited JSON).
4. **Map the TypeScript/Python types** to Go structs for the message protocol.

The protocol is not formally documented as a specification — it is defined implicitly by the SDK source code. A Go port would need to reverse-engineer / replicate the message format from the SDK source.

### Option 2: Embed Node.js or Python with the SDK

Shell out to `node` or `python` with the SDK installed. This adds a heavy runtime dependency but avoids reverse-engineering the protocol.

### Recommendation

Option 1 is preferable for `agynd` since it avoids Node.js/Python runtime dependencies in the agent container. The Codex app-server JSON-RPC v2 protocol is particularly well-suited for Go since JSON-RPC is a well-defined standard.

## SDK Capabilities Comparison

### Core Lifecycle

| Capability | Codex SDK (TS) | Codex SDK (Python) | Claude Code SDK (TS) | Claude Code SDK (Python) |
|---|---|---|---|---|
| **Create client** | `new Codex(options?)` | `Codex()` / `AsyncCodex()` | — (uses `query()` function) | — (uses `query()` function or `ClaudeSDKClient()`) |
| **Start session** | `codex.startThread(options?)` → `Thread` | `codex.thread_start(model=...)` → `Thread` | `query({prompt, options})` → `Query` (async generator) | `query(prompt=..., options=...)` → `AsyncIterator` |
| **Send prompt (buffered)** | `thread.run(prompt)` → `Turn` | `thread.turn(TextInput(...)).run()` → `Turn` | Iterate `query()` to completion → `SDKResultMessage` | Iterate `query()` to completion |
| **Send prompt (streaming)** | `thread.runStreamed(prompt)` → `{events}` | `thread.turn(...).stream()` → events | Same `query()` — always async generator | Same `query()` — always async iterator |
| **Resume session** | `codex.resumeThread(threadId)` | — | `options.resume = sessionId` | `ClaudeAgentOptions(resume=sessionId)` |
| **Multi-turn (client)** | — | — | Streaming input via `AsyncGenerator` | `ClaudeSDKClient`: `client.query()` / `client.receive_response()` |
| **Cancel / abort** | `AbortSignal` via `options.signal` | — | `query.interrupt()` / `AbortController` | `ClaudeSDKClient.interrupt()` |

### Session / Thread Management

| Capability | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **Session persistence** | Automatic (`~/.codex/sessions`); resume by thread ID | Automatic (`~/.claude/projects/`); resume by session ID |
| **V2 Session API (TS)** | — (single API) | `unstable_v2_createSession(options)` → `SDKSession` |
| **V2 resume (TS)** | — | `unstable_v2_resumeSession(sessionId, options)` |
| **V2 one-shot (TS)** | — | `unstable_v2_prompt(message, options)` → `SDKResultMessage` |
| **Interactive client (Python)** | — | `ClaudeSDKClient` — `query()`, `send()`, `receive_response()` |
| **Session close** | Context manager (`with Codex()`) | `session.close()` / `async with ClaudeSDKClient()` |
| **Fork session** | — | `options.forkSession = true` |
| **File checkpointing (Python)** | — | `ClaudeSDKClient.rewind_files(message_id)` |

### Configuration

| Capability | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **Working directory** | `startThread({workingDirectory})` / `thread_start(...)` | `options.cwd` / `ClaudeAgentOptions(cwd=...)` |
| **Model selection** | Via CLI config / environment / `thread_start(model=...)` | `options.model` / `ClaudeAgentOptions(model=...)` |
| **System prompt** | Via CLI config | `options.systemPrompt` / `ClaudeAgentOptions(system_prompt=...)` |
| **Environment variables** | `new Codex({env})` | `options.env` / `ClaudeAgentOptions(env=...)` |
| **Config overrides** | `new Codex({config})` — flattened TOML keys | `options.extraArgs` |
| **Max turns** | Via CLI config | `options.maxTurns` / `ClaudeAgentOptions(max_turns=...)` |
| **Max budget** | Via CLI config | `options.maxBudgetUsd` / `ClaudeAgentOptions(max_budget_usd=...)` |
| **Max thinking tokens** | Via CLI config | `options.maxThinkingTokens` / `ClaudeAgentOptions(max_thinking_tokens=...)` |
| **Skip git repo check** | `startThread({skipGitRepoCheck: true})` | — |
| **Additional directories** | — | `options.additionalDirectories` |
| **Settings sources** | — | `options.settingSources` / `ClaudeAgentOptions(setting_sources=...)` |

### Output / Results

| Capability | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **Final text response** | `turn.finalResponse` / `turn.status` | `resultMessage.result` (when `subtype === 'success'`) |
| **Structured output (JSON schema)** | `thread.run(prompt, {outputSchema})` | `options.outputFormat = {type: 'json_schema', schema}` |
| **Structured output result** | `turn.finalResponse` (parsed) | `resultMessage.structured_output` |
| **Turn items / events** | `turn.items` / streaming events | Stream of `SDKMessage` / Python message objects |
| **Usage / cost tracking** | `event.usage` (in streaming) | `resultMessage.total_cost_usd`, `resultMessage.usage`, `resultMessage.modelUsage` |
| **Duration tracking** | — | `resultMessage.duration_ms`, `resultMessage.duration_api_ms` |
| **Error types** | Thrown exceptions | `resultMessage.subtype`: `error_during_execution`, `error_max_turns`, `error_max_budget_usd` |
| **Image input** | `thread.run([{type: "local_image", path}])` | — (via tool use / prompt content) |

### Tool and Permission Control

| Capability | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **Tool allow-list** | — (controlled via CLI config) | `options.allowedTools` / `ClaudeAgentOptions(allowed_tools=[...])` |
| **Tool deny-list** | — | `options.disallowedTools` / `ClaudeAgentOptions(disallowed_tools=[...])` |
| **All built-in tools preset** | — | `{type: 'preset', preset: 'claude_code'}` |
| **Permission modes** | — (CLI `--approval` flag) | `permissionMode`: `default`, `acceptEdits`, `bypassPermissions`, `plan`, `dontAsk` |
| **Custom permission handler** | — | `options.canUseTool` (TS) / `ClaudeAgentOptions(can_use_tool=...)` (Python) |
| **Permission via MCP tool** | — | `options.permissionPromptToolName` |
| **Sandbox settings** | Via CLI config | `options.sandbox` / `ClaudeAgentOptions(sandbox=...)` |

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
| **MCP server config** | Via CLI config file | `options.mcpServers` / `ClaudeAgentOptions(mcp_servers=...)` |
| **In-process MCP tools (Python)** | — | `@tool` decorator → in-process MCP server (Python only) |
| **Strict MCP config** | — | `options.strictMcpConfig` |
| **List MCP resources** | — | `ListMcpResources` tool |
| **Read MCP resource** | — | `ReadMcpResource` tool |
| **MCP server status** | — | `query.mcpServerStatus()` |

### Hooks and Extensibility

| Capability | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **Hook system** | — | `options.hooks` (TS) / `ClaudeSDKClient` hooks (Python) |
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
| **Interrupt execution** | Abort via signal | `query.interrupt()` (TS) / `client.interrupt()` (Python) |
| **Change permission mode** | — | `query.setPermissionMode(mode)` / `client.set_permission_mode(mode)` |
| **Switch model** | — | `query.setModel(model)` / `client.set_model(model)` |
| **Set thinking tokens** | — | `query.setMaxThinkingTokens(n)` |
| **List available commands** | — | `query.supportedCommands()` |
| **List available models** | — | `query.supportedModels()` |
| **Get account info** | — | `query.accountInfo()` |
| **Rewind files** | — | `client.rewind_files(message_id)` (Python) |

### Message Types (Streaming)

| Message Type | Codex SDK | Claude Code Agent SDK |
|---|---|---|
| **Item completed** | `event.type === "item.completed"` | `msg.type === 'assistant'` / `AssistantMessage` |
| **Turn completed** | `event.type === "turn.completed"` | `msg.type === 'result'` / `ResultMessage` |
| **System / init** | — | `msg.type === 'system'` (subtype: `init`) / `SystemMessage` |
| **Tool progress** | — (via items) | `msg.type === 'tool_progress'` |
| **Auth status** | — | `msg.type === 'auth_status'` |
| **Partial / streaming** | — (via `runStreamed` events) | `msg.type === 'partial_assistant'` (if `includePartialMessages`) |
| **Compact boundary** | — | `msg.type === 'compact_boundary'` |
| **Hook response** | — | `msg.type === 'hook_response'` |

## Key Differences Summary

1. **API shape**: Codex uses an object-oriented model (`Codex` → `Thread` → `run()`) in both TS and Python. Claude Code uses a functional model (`query()` → async generator) plus `ClaudeSDKClient` for stateful sessions in Python.

2. **Feature surface**: Claude Code Agent SDK exposes significantly more control — custom permissions, hooks, sub-agents, tool allow/deny lists, sandbox configuration, budget limits, MCP server management, in-process custom tools (Python), and runtime query methods. Codex SDK is deliberately minimal: create thread, run prompt, get result.

3. **Streaming model**: Codex offers two modes (`run()` for buffered, `runStreamed()` for streaming). Claude Code always streams via async generator/iterator with a richer set of message types.

4. **Python maturity**: Claude Code Python SDK is stable and published on PyPI. Codex Python SDK is experimental and uses a different protocol (JSON-RPC v2 via `app-server`) compared to the TypeScript SDK.

5. **Underlying mechanism**: Both spawn their respective CLI binary and communicate over stdio using structured messages. This means a Go port follows the same pattern for both: spawn process, write/read structured messages. The exact protocol differs (JSONL vs JSON-RPC v2).

6. **Custom tools**: Claude Code Python SDK supports defining custom tools as Python functions via `@tool` decorator (in-process MCP servers). Codex SDK does not expose this.

## References

- Codex SDK TypeScript source: https://github.com/openai/codex/tree/main/sdk/typescript
- Codex SDK Python source: https://github.com/openai/codex/tree/main/sdk/python
- Codex SDK docs: https://developers.openai.com/codex/sdk/
- Claude Code Agent SDK TypeScript on npm: https://www.npmjs.com/package/@anthropic-ai/claude-agent-sdk
- Claude Code Agent SDK Python on PyPI: https://pypi.org/project/claude-agent-sdk/
- Claude Code Agent SDK Python source: https://github.com/anthropics/claude-agent-sdk-python
- Claude Code Agent SDK docs: https://platform.claude.com/docs/en/agent-sdk/overview
- Claude Code Agent SDK Python reference: https://platform.claude.com/docs/en/agent-sdk/python
