# Transcript-Sourced Tracing

## Target

- [Tracing — Overview](../architecture/tracing.md#overview)
- [Tracing — agynd Tracing Proxy](../architecture/tracing.md#agynd-tracing-proxy)
- [Tracing — Agent CLI Configuration](../architecture/tracing.md#agent-cli-configuration)
- [Tracing — Proxy Behavior](../architecture/tracing.md#proxy-behavior)
- [agynd — Environment Preparation](../architecture/agynd-cli.md#3-environment-preparation)

## Delta

Tracing is specified to capture **the full LLM call context — the complete request body** for each call, and the run view is built on four span names that carry it: `invocation.message`, `llm.call`, `tool.execution`, `summarization`. None of that arrives. A trace of a real turn is forty spans of `busy_ns`, `code.file.path` and `thread.name` — the Rust `tracing` crate's own instrumentation, exported verbatim — and the run view renders none of it, because none of it is what the view was built to read.

The cause is that the agent CLI's OTel export is the wrong source. Of 1351 spans collected from real turns, **zero** carry any `gen_ai.*` attribute. `codex.api_request` reports an endpoint, a model, a status and a duration; `codex.user_prompt` reports `prompt_length` — a number, not a prompt. `response.output_text.delta` fires once per streamed chunk and carries a duration and nothing else. The signal is metadata by construction: it describes that a call happened, never what was said. No configuration recovers the content, because the content was never put on the wire.

The agent CLI writes it to disk instead. Every turn is appended to a session transcript — Codex's rollout JSONL, Claude Code's conversation transcript — holding the prompt, the assistant's reply, each tool call with its input and output, and the token counts. The record the product needs already exists, complete, in the container that produced it.

So the span producer moves. It stops being the CLI's OTel SDK and becomes a **lifecycle hook that reads the transcript**.

### The hook is the producer

Each agent CLI runs a hook after every turn. The hook reads the transcript, reconstructs the turn, and posts it to `agynd`, which converts it into spans and forwards them as it does today.

| Agent CLI | Hook | Transcript |
|-----------|------|------------|
| **Codex** | `Stop`, configured in `config.toml`; the rollout path arrives on stdin | rollout JSONL |
| **Claude Code** | `Stop` and `SessionEnd`, configured in `~/.claude/settings.json` | conversation transcript |

`agynd` already writes both files during [environment preparation](../architecture/agynd-cli.md#3-environment-preparation), so the hook is installed where the CLI is already configured. It reaches `agynd` over loopback, the same boundary the OTLP endpoint uses, and `agynd` remains the only process holding platform context.

A turn becomes one `invocation.message` with the prompt, an `llm.call` per model step carrying model, usage and the full request context, and a `tool.execution` per tool call carrying its input and output. Subagent turns nest under the turn that spawned them. This is the vocabulary the run view already renders; nothing downstream changes.

### Consequences

**The proxy stops being pass-through.** It gains a second intake — turns from the hook — alongside the OTLP endpoint it already serves. Attribution is unchanged: the same `agyn.agent_instance.id`, `agyn.thread.id`, `agyn.thread.message.id` and `agyn.workload.id` are injected on the way out, and the Tracing service verifies them exactly as before. A hook that can post turns can assert no more than an agent CLI that can export spans.

**A turn is uploaded once.** The transcript is cumulative and a resumed session replays it, so the hook records which turns it has already sent — Codex's rollout carries a sidecar beside it, Claude Code's state lives with its settings — and sends only what is new.

**A tracing failure never reaches the agent.** The hook runs inside the turn's lifecycle, so it fails open: an error is logged and swallowed. Tracing is an optional dependency, and an unreachable Tracing service must not end a turn that has otherwise succeeded.

**The OTel export is no longer the source of agent activity.** The CLI's own spans describe the framework's internals — nesting ten deep, one span per streamed chunk — and answer no question the run view asks. They stop being collected.

### Version floor

The hook contract is what makes this possible, and it is recent: the Codex `Stop` hook passes the rollout path on stdin from **0.128**. The pinned runtime is older and moves to a version that carries it. Claude Code's hooks carry no such floor.
