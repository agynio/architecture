# LLM Provider Custom Auth Headers

## Target

- [Providers, Models, and Secrets — LLM Provider](../architecture/providers.md#llm-provider)
- [LLM Service — ResolveModel](../architecture/llm.md#resolvemodel)
- [LLM Proxy — Protocols](../architecture/llm-proxy.md#protocols)
- [Console — LLM Providers and Models](../product/console/console.md#llm-providers-and-models)
- [Terraform Provider — LLM Provider Resource](../architecture/operations/terraform-provider.md#llm-provider-resource)

## Delta

The `LLMProvider` resource and its consumers (LLM service, LLM Proxy, Console, Terraform provider) need to support authentication beyond a single bearer-style token so that providers requiring `x-api-key` (Anthropic) or non-standard header combinations (internal gateways, OpenRouter-style routing headers, providers whose token format is not `Bearer <token>`) can be configured.

- **LLM service / providers schema** — `authMethod` currently enumerates `bearer` and `x_api_key` only, and the credential payload is a single `token` string. A third value `custom_headers` is needed, carrying an arbitrary `headers` map<string,string> that the LLM Proxy injects verbatim on every forwarded request.
- **`ResolveModel` response** — only returns `token` and `auth_method`. It must also return `headers` so the LLM Proxy can forward the configured custom-header set. Reserved hop-by-hop / routing header names (`Host`, `Content-Length`, `Connection`, `Transfer-Encoding`) must be rejected on Create/Update so the proxy is never asked to set them.
- **LLM Proxy** — currently switches between `Authorization: Bearer <token>` and `x-api-key: <token>`. It must add a `custom_headers` branch that copies each key/value from the resolved `headers` map onto the outbound request, with the same precedence rules as the named methods (proxy-injected auth overrides any caller-provided value for the same header).
- **Console** — the LLM Provider create/edit form exposes only a single `Token` field. It needs an auth-method selector (`Bearer` / `x-api-key` / `Custom headers`) that swaps the credential editor between a single masked token input and an editable header-row list (key + value, value masked with reveal-on-click). The provider list and detail views must show the selected auth method.
- **Terraform provider** — `agyn_llm_provider` and `agyn_llm_model` resources are not yet exposed by `agynio/terraform-provider-agyn`. Providers and models are only manageable via the Console / Gateway today. The new resources must expose `auth_method`, `token`, and `headers` with plan-time validation of the `token`/`headers` invariants (exactly one of the two, by auth method).

## Acceptance Signal

- A `custom_headers` provider can be created via the Gateway, the Console form, and the Terraform provider with a non-empty `headers` map; `token` is rejected when set. Conversely, `bearer` / `x_api_key` providers reject a non-empty `headers` map.
- Reserved header names are rejected with a clear validation error on Create/Update from all three entry points.
- `ResolveModel` returns the `headers` map for `custom_headers` providers and an empty map for `bearer` / `x_api_key` providers; `token` is returned for the latter two and empty for the former.
- An end-to-end LLM call through the LLM Proxy against a `custom_headers` Anthropic provider configured with `{"x-api-key": "<key>", "anthropic-version": "2023-06-01"}` succeeds, with the proxy injecting both headers on the forwarded request.
- The Console model-test dialog succeeds against a model whose provider uses `custom_headers`.
- `terraform plan` with conflicting `token` + `custom_headers`, or empty `headers` + `custom_headers`, fails at plan time (not apply).

## Notes

The `headers` map is stored encrypted at rest in the LLM service database using the same mechanism as the existing `token` field. The map is sensitive in Terraform state and is masked in Gateway / Console responses (drift on header values is intentionally not detected after creation, mirroring `token`).

`custom_headers` does not replace `bearer` / `x_api_key`. The named methods remain the ergonomic path for the common case (one credential header, well-known format) and keep the Console UI simple for typical OpenAI / Anthropic onboarding. `custom_headers` is the escape hatch for everything else.
