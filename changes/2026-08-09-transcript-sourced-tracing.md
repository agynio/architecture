# Transcript-Sourced Tracing

## Target

- [Tracing — Tracing Plugins](../architecture/tracing.md#tracing-plugins)
- [Tracing — agynd Tracing Proxy](../architecture/tracing.md#agynd-tracing-proxy)
- [Tracing — Proxy Behavior](../architecture/tracing.md#proxy-behavior)
- [agynd — Environment Preparation](../architecture/agynd-cli.md#3-environment-preparation)

## Delta

Tracing is specified to capture **the full LLM call context — the complete request body** for each call, and the run view is built on four span names that carry it: `invocation.message`, `llm.call`, `tool.execution`, `summarization`. None of that arrives. A trace of a real turn is forty spans of `busy_ns`, `code.file.path` and `thread.name` — the Rust `tracing` crate's own instrumentation, exported verbatim — and the run view renders none of it, because none of it is what the view was built to read.

The cause is that the agent CLI's OTel export is the wrong source. Of 1351 spans collected from real turns, **zero** carry any `gen_ai.*` attribute. `codex.api_request` reports an endpoint, a model, a status and a duration; `codex.user_prompt` reports `prompt_length` — a number, not a prompt. `response.output_text.delta` fires once per streamed chunk and carries a duration and nothing else. The signal is metadata by construction: it describes that a call happened, never what was said. No configuration recovers the content, because the content was never put on the wire.

The agent CLI writes it to disk instead. Every turn is appended to a session transcript — Codex's rollout JSONL, Claude Code's conversation transcript — holding the prompt, the assistant's reply, each tool call with its input and output, and the token counts. The record the product needs already exists, complete, in the container that produced it.

### What must change

The span producer stops being the CLI's OTel SDK. The [tracing plugin](../architecture/tracing.md#tracing-plugins) is written for each agent CLI, ships with the agent runtime image, and `agynd` registers its hook while writing the CLI's config — the file it already writes for the LLM endpoint and MCP servers. The proxy gains the intake that receives what the plugin posts.

The CLI's own OTel export stops being collected: it describes the framework's internals and answers no question the run view asks.

### Version floor

The hook contract is what makes this possible, and it is recent: the Codex `Stop` hook passes the rollout path on stdin from **0.128**. The pinned runtime is older and moves to a version that carries it. Claude Code's hooks carry no such floor.
