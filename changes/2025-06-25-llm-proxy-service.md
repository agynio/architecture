# LLM Proxy Service

## Target

- [LLM Proxy](../architecture/llm-proxy.md)
- [LLM Service](../architecture/llm.md)
- [Gateway](../architecture/gateway.md)
- [OpenZiti Integration](../architecture/openziti.md)

## Delta

The LLM Proxy service (`agynio/llm-proxy`) does not exist. The following are needed:

- **LLM Proxy service**: new Go service exposing `POST /v1/responses` (OpenAI Responses API format) with OpenZiti and API token authentication, org-level authorization, model resolution via LLM service gRPC, and request forwarding to external LLM providers with streaming support.
- **LLM service `ResolveModel` method**: new gRPC method returning provider endpoint, token, remote model name, and organization ID for a given model ID.
- **Gateway `LLMGateway` removal**: remove the `LLMGateway` ConnectRPC service and `agynio/api/gateway/v1/llm.proto`. LLM proxy traffic no longer goes through the Gateway.
- **OpenZiti bootstrap**: new `llm-proxy` service, `llm-proxy-bind` and `agents-dial-llm-proxy` static policies, `llm-proxy-hosts` role attribute.
- **Ingress**: Istio VirtualService for `llm.agyn.dev` → `llm-proxy:8080`.
- **Agent environment**: `agynd` must configure agents with the LLM Proxy endpoint (OpenZiti service address or public URL) instead of a direct LLM service address.

## Acceptance Signal

- Agents (Codex CLI, agn) successfully make OpenAI Responses API calls through the LLM Proxy, with model resolution and streaming working end-to-end.
- The `LLMGateway` proto service and handler are removed from the Gateway.
- OpenZiti policies allow agents to dial the `llm-proxy` service.

## Notes

- The LLM Proxy authenticates independently from the Gateway (OpenZiti + API tokens).
- Authorization is org-level only — any identity in an org can use any model in that org.
- The LLM service retains provider/model CRUD responsibility; proxy responsibility moves entirely to the new service.
