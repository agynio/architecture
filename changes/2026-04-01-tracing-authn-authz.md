# Tracing Authentication and Authorization

## Target

- [Tracing — Authentication and Authorization](../architecture/tracing.md#authentication-and-authorization)
- [Tracing — agynd Tracing Proxy](../architecture/tracing.md#agynd-tracing-proxy)
- [Agents Service — Internal API](../architecture/agents-service.md#internal-api)
- [OpenZiti — Static Policies](../architecture/openziti.md#static-policies) (tracing entries)

## Delta

- The Tracing service does not participate in the OpenZiti overlay. No `tracing` OpenZiti service exists, no self-enrollment, no identity extraction from connections.
- No ingestion authentication — the `TraceService/Export` endpoint accepts unauthenticated requests.
- No attribute injection — the Tracing service does not derive or overwrite `agyn.identity.id`, `agyn.agent.id`, `agyn.organization.id` on stored spans.
- No thread verification — `agyn.thread.id` is not verified against the Authorization service.
- `agynd` does not run an OTLP tracing proxy. Agent CLIs have no local endpoint to export spans to.
- `agynd` does not inject `agyn.thread.id` as a resource attribute on exported spans.
- The Agents service does not expose `ResolveAgentIdentity`.
- The `agents-dial-tracing` and `tracing-bind` static policies do not exist in the OpenZiti configuration.
- The `tracing-hosts` role attribute is not provisioned.
- No LRU caching for identity chain resolution or thread authorization in the Tracing service.
- No query authorization — `ListSpans`, `GetTrace`, `GetSpan` do not enforce organization membership.

## Acceptance Signal

- Agent CLI spans exported via `agynd` tracing proxy arrive at the Tracing service with `agyn.thread.id` injected by `agynd` and `agyn.identity.id`, `agyn.agent.id`, `agyn.organization.id` injected by the Tracing service from verified OpenZiti identity.
- Spans with invalid `agyn.thread.id` (failing Authorization check) are rejected.
- `ResolveAgentIdentity` resolves identity→agent→org for all agent types.
- Query API enforces organization membership.

## Notes

- LLM Proxy is unaffected — agent LLM traffic continues to flow via `llm-proxy.ziti` directly, not through `agynd`.
- `agynd` does not produce spans itself; it only proxies and enriches.
- Query authorization (org-scoped visibility) is lower priority than ingestion auth but included in the spec.
