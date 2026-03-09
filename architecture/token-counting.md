# Token Counting

## Overview

A dedicated service that counts tokens for LLM messages. It receives an array of messages and returns the token count for each message.

The current platform estimates tokens using `text.length / 4`, which is inaccurate for text and does not work at all for media (images, files). This service replaces that heuristic with accurate per-message token counts, using the actual tokenizer for the target model.

## Motivation

- **litellm's `input_tokens`** was evaluated and does not provide reliable pre-call token counts.
- Summarization needs to know token counts **before** the LLM call to decide whether to summarize.
- Media files (images, documents) have token costs that cannot be estimated from byte size or string length.

## Interface

The service exposes a single method.

### CountTokens

**Request:**

| Field | Type | Description |
|-------|------|-------------|
| `model` | string | Target model identifier (e.g., `gpt-4o`, `claude-sonnet-4-20250514`) |
| `messages` | list of Message | Messages to count tokens for |

Each message in the array is a serialized OpenAI Responses API input/output item — the same shape produced by `toPlain()` on the platform's LLM message types:

| Message type | Responses API shape |
|-------------|-------------------|
| `HumanMessage` | `ResponseInputItem.Message` (role: `user`) |
| `SystemMessage` | `ResponseInputItem.Message` (role: `system`) |
| `AIMessage` | `ResponseOutputMessage` (role: `assistant`) |
| `ToolCallMessage` | `ResponseFunctionToolCall` |
| `ToolCallOutputMessage` | `ResponseInputItem.FunctionCallOutput` |
| `ResponseMessage` | `{ output: ResponseOutput[] }` — a wrapper containing multiple output items |

Currently only OpenAI Responses API format is supported. The service is designed to be extended with other formats in the future.

**Response:**

| Field | Type | Description |
|-------|------|-------------|
| `tokens` | list of integer | Token count per message, in the same order as the input |

The response array has the same length as the input `messages` array. Each element is the token count for the corresponding message.

### Example

**Request:**
```json
{
  "model": "gpt-4o",
  "messages": [
    { "type": "message", "role": "system", "content": [{ "type": "input_text", "text": "You are a helpful assistant." }] },
    { "type": "message", "role": "user", "content": [{ "type": "input_text", "text": "Hello" }] },
    { "type": "message", "role": "user", "content": [
      { "type": "input_text", "text": "What is in this image?" },
      { "type": "input_image", "image_url": "https://..." }
    ]}
  ]
}
```

**Response:**
```json
{
  "tokens": [12, 4, 1028]
}
```

## Classification

The Token Counting service is a **data plane** service — it is called on the hot path during agent execution, before each LLM call.

## Integration with Summarization

The summarization reducer currently calls `countTokensFromMessages` which uses the `text.length / 4` heuristic. This internal method is replaced with a call to the Token Counting service:

1. Before the LLM call, the summarization reducer sends the current message array to the Token Counting service.
2. The service returns per-message token counts.
3. The reducer sums the counts and compares against `maxTokens` to decide whether to summarize.
4. If summarization is needed, per-message counts are used for the head/tail split (`keepTokens` budget).
