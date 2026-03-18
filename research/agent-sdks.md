# Agent SDKs Research

Research on programmatic SDKs for Codex and Claude Code — the two 3rd-party agent CLIs that `agynd` needs to control.

## Overview

Both Codex and Claude Code provide TypeScript/Node.js SDKs for programmatic control. **Neither has a Go SDK.** Both SDKs work by spawning their respective CLI as a child process and exchanging messages over stdin/stdout (JSONL protocol). This is important for our Go integration strategy.

| Aspect | Codex SDK | Claude Code Agent SDK |
|--------|-----------|----------------------|
| Package | `@openai/codex-sdk` | `@anthropic-ai/claude-agent-sdk` |
| Language | TypeScript (Node.js 18+) | TypeScript (Node.js 18+) |
| Go SDK | None | None |
| Mechanism | Spawns `codex` CLI, JSONL over stdin/stdout | Spawns `claude` CLI, JSONL over stdin/stdout |
| Latest version (at time of research) | Part of `@openai/codex` monorepo | `0.2.x` on npm |
| License | Apache-2.0 | Commercial (Anthropic ToS) |

## Go Integration Strategy

Since both SDKs are thin wrappers that spawn CLI processes and exchange JSONL over stdio, porting to Go is feasible. The work involves:

1. **Spawn the CLI binary** as a child process from Go (`os/exec`).
2. **Write JSONL commands** to the process stdin.
3. **Read JSONL events** from the process stdout (line-delimited JSON).
4. **Map the TypeScript types** to Go structs for the message protocol.

The protocol is not formally documented as a specification — it is defined implicitly by the SDK source code. A Go port would need to reverse-engineer / replicate the JSONL message format from the TypeScript SDK source.

An alternative is to embed a Node.js runtime or shell out to `node -e` with the SDK, but this adds a heavy dependency.

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

4. **Underlying mechanism**: Both spawn their respective CLI binary and communicate over stdin/stdout using JSONL. This means a Go port follows the same pattern for both: spawn process, write/read JSONL lines.

## References

- Codex SDK source: https://github.com/openai/codex/tree/main/sdk/typescript
- Codex SDK docs: https://developers.openai.com/codex/sdk/
- Claude Code Agent SDK on npm: https://www.npmjs.com/package/@anthropic-ai/claude-agent-sdk
- Claude Code Agent SDK docs: https://platform.claude.com/docs/en/agent-sdk/overview
