# Claude Code Agent Support

## Target

- [Architecture: LLM Proxy](../architecture/llm-proxy.md)
- [Architecture: LLM Service](../architecture/llm.md)
- [Architecture: Providers, Models, and Secrets — LLM Provider](../architecture/providers.md#llm-provider)
- [Architecture: agynd-cli — LLM Endpoint Configuration](../architecture/agynd-cli.md#llm-endpoint-configuration)
- [Architecture: agynd-cli — Agent Communication Protocol](../architecture/agynd-cli.md#agent-communication-protocol)
- [Architecture: Agent Init Container](../architecture/agent-init.md)

## Delta

Claude Code cannot run as an agent on the platform. The following are not implemented:

### LLM Proxy — Anthropic Messages API endpoint

- `POST /v1/messages` endpoint does not exist. The LLM Proxy only serves `POST /v1/responses` (OpenAI Responses API).
- Anthropic Messages API request/response forwarding is not implemented (different request schema, different SSE streaming event types, `anthropic-version` header forwarding).
- Caller authentication via `x-api-key` header is not implemented (the proxy only reads `Authorization: Bearer`).
- Wire protocol validation (caller endpoint must match provider `wire_api`) is not implemented.

### LLM Service — Protocol-aware model resolution

- `ResolveModel` does not return `wire_api` or `auth_method`. It returns `endpoint`, `token`, `remote_name`, `organization_id` only.
- The LLM proto (`agynio/api`) does not have `wire_api` or `auth_method` fields on the `ResolveModelResponse` message.

### LLM Provider — `wire_api` and `authMethod` extension

- The LLM Provider resource has no `wire_api` field. All providers are assumed to speak OpenAI Responses API.
- The `authMethod` enum only supports `bearer`. `x_api_key` (for Anthropic's `x-api-key` header) is not supported.
- The LLM Provider proto (`agynio/api`) does not have a `wire_api` field on the `LLMProvider` message.
- The Agents service (`agynio/agents`) does not store or validate `wire_api` on LLM Provider resources.
- The Terraform provider (`agynio/terraform-provider-agyn`) does not expose `wire_api` on the `agyn_llm_provider` resource.

### agynd — Claude Code environment preparation

- `agynd` does not write `~/.claude/settings.json`. Claude Code LLM endpoint configuration (via `ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN` in the `env` block) is not implemented.
- `agynd` does not configure Claude Code permissions (the `permissions` block granting all built-in tools).
- `agynd` does not set `DISABLE_AUTOUPDATER` or `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` for Claude Code.
- Skills placement for Claude Code (filesystem path conventions for `CLAUDE.md` or equivalent) is not implemented.
- MCP server configuration for Claude Code (writing `~/.claude/settings.json` `mcpServers` block, or using `--mcp-config` flag) is not implemented.

### agynd — `claude-sdk-go` integration

- The `claude-sdk-go` repository (`agynio/claude-sdk-go`) does not exist.
- `agynd` returns `unsupported` at runtime when `sdk: claude` is specified in `config.json`.
- The Claude Code JSONL subprocess protocol (spawn `claude --output-format stream-json --input-format stream-json --verbose`, parse NDJSON messages discriminated by `type` field) is not implemented in any Go module.

### Agent init container — `agent-init-claude`

- The `agynio/agent-init-claude` repository does not exist.
- The init container image `ghcr.io/agynio/agent-init-claude:<version>` is not built or published.
- Claude Code native binary (`linux-x64-musl` variant) download and packaging in the Dockerfile is not implemented.
- `libgcc`/`libstdc++` bundling for the Claude Code binary on Alpine-based user images is not implemented.
- `config.json` with `{ "sdk": "claude", "bin": "/agyn-bin/claude" }` does not exist.
- `versions.env` pinning `AGYND_VERSION` and `CLAUDE_VERSION` does not exist.
- CI workflow for building and publishing the image does not exist.

## Acceptance Signal

- An agent with `init_image: ghcr.io/agynio/agent-init-claude:<version>` starts successfully.
- Claude Code inside the agent container connects to the LLM Proxy via `POST /v1/messages` and receives streaming responses.
- An LLM Provider with `wire_api: anthropic_messages` and `authMethod: x_api_key` can be created and used to resolve models.
- Claude Code has full tool permissions (no interactive prompts) via `settings.json` permissions block.
- Claude Code has auto-updater and non-essential traffic disabled.
- `claude-sdk-go` spawns the Claude Code subprocess, sends user messages, and receives NDJSON responses.
- `agynd` writes `~/.claude/settings.json` with the correct `env` and `permissions` blocks before spawning Claude Code.
- MCP tool servers are accessible to Claude Code via MCP configuration.
- The `agyn_llm_provider` Terraform resource supports the `wire_api` field.

## Notes

- `claude-sdk-go` protocol spec lives in the `agynio/claude-sdk-go` repository, not in architecture docs. The protocol is reverse-engineered from Anthropic's Python and TypeScript SDK sources and has no formal specification.
- Token counting changes are out of scope. Claude Code handles its own token counting internally.
- The LLM Proxy does not translate between wire protocols. A `POST /v1/messages` request is only valid when the resolved provider has `wire_api: anthropic_messages`.
- Claude Code's native binary (`linux-x64-musl`) is not fully static — it dynamically links `libgcc` and `libstdc++`. The init container image must bundle these libraries (from Alpine's `libgcc` and `libstdc++` packages) and ensure they are available at `/agyn-bin/lib/` or equivalent for the binary to find at runtime.
- Claude Code permission management uses `settings.json` `permissions.allow` block to grant all tool permissions. The platform provides security isolation at the container level, so the agent can perform any action.
