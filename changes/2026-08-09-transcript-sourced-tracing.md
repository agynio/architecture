# Transcript-Sourced Tracing

## Target

- [Tracing — Span Producers](../architecture/tracing.md#span-producers)
- [Tracing — The Trace Hook](../architecture/tracing.md#the-trace-hook)
- [Tracing — Attribute Injection and Verification](../architecture/tracing.md#attribute-injection-and-verification)
- [agynd — Environment Preparation](../architecture/agynd-cli.md#3-environment-preparation)

## Delta

Tracing is specified to capture **the full LLM call context — the complete request body** for each call, and the run view is built on four span names that carry it: `invocation.message`, `llm.call`, `tool.execution`, `summarization`. None of that arrives. A trace of a real turn is forty spans of `busy_ns`, `code.file.path` and `thread.name` — the Rust `tracing` crate's own instrumentation, exported verbatim — and the run view renders none of it, because none of it is what the view was built to read.

The cause is that the agent CLI's OTel export is the wrong source. Of 1351 spans collected from real turns, **zero** carry any `gen_ai.*` attribute. `codex.api_request` reports an endpoint, a model, a status and a duration; `codex.user_prompt` reports `prompt_length` — a number, not a prompt. `response.output_text.delta` fires once per streamed chunk and carries a duration and nothing else. The signal is metadata by construction: it describes that a call happened, never what was said. No configuration recovers the content, because the content was never put on the wire.

The agent CLI writes it to disk instead. Every turn is appended to a session transcript — Codex's rollout JSONL, Claude Code's conversation transcript — holding the prompt, the assistant's reply, each tool call with its input and output, and the token counts. The record the product needs already exists, complete, in the container that produced it.

### What must change

The span producer stops being the CLI's OTel SDK. The [trace hook](../architecture/tracing.md#the-trace-hook) reads the transcript instead, and `agynd` registers it while writing the CLI's config — the file it already writes for the LLM endpoint and MCP servers.

It is one binary rather than one plugin per CLI, and it ships with `agynd`. A hook is a command to run, so nothing about it is CLI-specific except the transcript it is handed; a per-CLI plugin would duplicate the turn model, the export and the delivery record — some 70% of it — across implementations, and let the span vocabulary drift between them with nothing to catch it.

**The tracing proxy is removed.** It existed to inject attribution onto spans passing through it, and there is nothing left for it to inject. The Tracing service derives identity from the connection, so a workload authenticating as its instance already *is* the attribution. Thread attribution goes with it: an instance serves an inbox drawn from many threads, and the per-turn value the proxy injected was documented as best-effort because there is no single thread a workload belongs to. `agynd` asserts the message on the `invocation.message` it emits, and nothing asserts a thread.

Producers export to `tracing.ziti` directly, each carrying its own identity. `agynd` opens a trace for the wake cycle and hands the id to the hook it registers, so a turn's spans and the message that opened it land together. The id is derived from `WORKLOAD_ID` rather than drawn, so an `agynd` restarting in the pod reopens the trace it was already writing.

The CLI's own OTel export stops being collected: it describes the framework's internals and answers no question the run view asks.

### Version floor

The hook contract is what makes this possible, and it is recent: the Codex `Stop` hook passes the rollout path on stdin from **0.128**. The pinned runtime is older and moves to a version that carries it. Claude Code's hooks carry no such floor.
