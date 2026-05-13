# LLM Proxy Forwards Caller Headers

## Target

- [LLM Proxy — Header Forwarding](../architecture/llm-proxy.md#header-forwarding)

## Delta

The LLM Proxy currently forwards the request body to the provider but only the `anthropic-version` header is explicitly carried through; everything else the caller sets is dropped. Standard LLM client libraries inject feature-flag and versioning headers — `anthropic-beta` (prompt caching, MCP, extended thinking, etc.), `openai-beta`, additional `anthropic-version` values — and these features silently fail when the headers don't reach the provider.

The proxy must forward all caller request headers verbatim to the provider, except for three classes that are stripped:

- Hop-by-hop / routing headers (`Host`, `Content-Length`, `Connection`, `Transfer-Encoding`, `Keep-Alive`, `TE`, `Trailer`, `Upgrade`, `Proxy-*`) — recomputed by the proxy for the outbound request.
- Proxy-authentication headers (`Authorization`, `x-api-key`) — they authenticate the caller to the proxy; the proxy injects its own provider credentials per `authMethod`.
- Platform-internal headers (`x-agyn-*`, e.g., `x-agyn-thread-id`) — consumed by the proxy and not part of any provider API.

Proxy-injected headers (per `authMethod`, or entries from a provider's `custom_headers` map) continue to override any caller-supplied value for the same header name.

## Acceptance Signal

- A caller sends `POST /v1/messages` with `anthropic-beta: prompt-caching-2024-07-31`; the outbound request to the provider includes the same header and prompt caching takes effect.
- A caller sends `POST /v1/responses` with `openai-beta: <flag>`; the outbound request includes the header.
- Caller-supplied `Authorization`, `x-api-key`, and `x-agyn-thread-id` are not present on the outbound request — replaced by proxy-injected provider credentials and consumed internally, respectively.
- Hop-by-hop headers supplied by the caller do not appear on the outbound request.
- For a provider with `authMethod: custom_headers` whose `headers` map contains `anthropic-version`, the provider's value is used even when the caller supplies a different `anthropic-version` value.
