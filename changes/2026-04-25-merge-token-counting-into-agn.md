# Merge Token Counting into agn

## Target

- [Architecture: Token Counting](../architecture/token-counting.md)
- [Architecture: agn-cli — Architecture diagram](../architecture/agn-cli.md#architecture)
- [Architecture: agn-cli — Relationship to Other Components](../architecture/agn-cli.md#relationship-to-other-components)

## Delta

### agn

- `agn` has no embedded token counting package. The `Summarizer` uses an inline `DefaultTokenCounter` with a `rune_count / 4` heuristic.
- `agn` carries no in-process implementation of BPE tokenization (`o200k_base`), tile-based image counting, or PDF text/page-image estimation.
- The `agn` architecture diagram has no Token Counting node; the `Summarization → Token Counting` edge is not present in code.

### Token Counting service

A standalone Token Counting service exists today and must be removed end-to-end:

- `agynio/token-counting` repository / deployment / Kubernetes manifests.
- `TokenCountingGateway` ConnectRPC handler in the Gateway.
- `agynio/api/gateway/v1/token_counting.proto` and any internal `agynio/api/.../token_counting.proto` definitions.
- Authorization rules covering `TokenCountingGateway` operations.
- Service-level configuration (env vars, Helm values, Istio policies).

## Acceptance Signal

- No `agynio/token-counting` repository, deployment, gateway route, authorization rule, or proto.
- `agn` performs token counting in-process for text (BPE `o200k_base`), images (tile-based geometry), and PDFs (text + per-page image cost) — matching the strategies described in [Token Counting](../architecture/token-counting.md#counting-strategies).
- `agn`'s only data-plane dependencies are the LLM endpoint and (optional) tracing — no outbound call for token counting.

## Notes

- Rationale: every running agent has its own `agn` process. The principle is that the agent depends only on the LLM endpoint (with optional tracing). A separate token counting service violates this for a ~2MB BPE table and a Go-native PDF parser that fit comfortably in the binary. The cost of an extra network hop on every message and an extra service to operate exceeds the cost of embedding.
- The existing [2026-08-22-token-counting-accurate-tokenization.md](2026-08-22-token-counting-accurate-tokenization.md) describes adding accurate BPE/image/PDF counting. That delta still applies — accuracy is needed regardless of where the code lives. The integration target shifts from a gRPC client call to an embedded package; the counting strategies are unchanged.
