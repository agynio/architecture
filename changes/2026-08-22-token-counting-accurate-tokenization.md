# Token Counting: Accurate Tokenization

## Target

- [Token Counting — Counting Strategies](../architecture/token-counting.md#counting-strategies)
- [Token Counting — Message Parsing](../architecture/token-counting.md#message-parsing)
- [Token Counting — Dependencies](../architecture/token-counting.md#dependencies)

## Delta

### Token counting service

- The tokenizer uses a `rune_count / 4` heuristic for all content. It does not perform BPE tokenization.
- `tiktoken-go` is not a dependency. The `o200k_base` encoding is not used.
- `pdfcpu` is not a dependency. PDF text extraction and page dimension reading are not implemented.
- The service does not parse message JSON structure. It treats each message as an opaque byte payload and counts runes across the entire serialized JSON — including JSON syntax characters, field names, and non-text content (base64 image data, file data).
- No content-type routing exists. `input_text`, `input_image`, and `input_file` content parts are not identified or handled differently.
- Image token counting (tile-based geometry from pixel dimensions) is not implemented.
- File token counting (PDF text extraction + per-page image cost estimation) is not implemented.
- Image dimension decoding (reading image headers for width/height) is not implemented.
- Per-message overhead tokens (role markers, separators) are not accounted for.

### agn (agent CLI)

- `agn` does not call the Token Counting service. The `Summarizer` uses an inline `DefaultTokenCounter` with the same `rune_count / 4` heuristic.
- The `Summarizer.TokenCounter` interface accepts a pluggable function, but `buildAgent` never injects a Token Counting service client.
- Token counts stored in `MessageRecord.TokenCount` are based on the heuristic, not accurate BPE counts.

## Acceptance Signal

- The token counting service returns BPE-accurate counts for text content using `o200k_base`.
- The token counting service returns geometry-accurate counts for `input_image` content using the tile-based formula with gpt-5 constants (base=70, tile=140).
- The token counting service returns estimated counts for `input_file` content (PDF: extracted text tokens + per-page image tokens; non-PDF text files: extracted text tokens).
- All counting is performed locally with no external service dependencies.
- `agn` calls the Token Counting service instead of using the inline heuristic.

## Notes

- The gRPC interface (`CountTokens` request/response shape) does not change. Messages are already passed as raw JSON byte arrays — the service just needs to parse them instead of counting runes.
- Audio content is not in scope. The OpenAI Responses API does not yet support audio input. When it does, the token counting service can be extended.
- PDF per-page image token estimation uses 150 DPI as a conservative upper bound for OpenAI's undocumented internal rendering resolution. This slightly overestimates, which is safer for summarization budget decisions.
- The `agn` integration (replacing `DefaultTokenCounter` with a gRPC client call) is a separate code change in the `agn-cli` repo but part of the same delta.
