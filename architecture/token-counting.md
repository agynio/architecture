# Token Counting

## Overview

A dedicated service that counts tokens for LLM messages. It receives an array of messages and returns the token count for each message.

The current platform estimates tokens using `text.length / 4`, which is inaccurate for text and does not work at all for media (images, files). This service replaces that heuristic with accurate per-message token counts.

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
| `model` | enum | Target model. Supported values: `gpt-5` |
| `messages` | list of Message | Messages to count tokens for. Each message is a serialized OpenAI Responses API input/output item |

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
  "model": "gpt-5",
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

## Token Storage

The agent counts tokens once when it processes a message and persists the count in the Agent State service. The summarization reducer reads stored token counts from agent state rather than re-counting. It sums the per-message counts to decide whether to summarize (`maxTokens` threshold) and uses them for the head/tail split (`keepTokens` budget).
