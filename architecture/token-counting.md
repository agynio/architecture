# Token Counting

## Overview

A dedicated service that counts tokens for LLM messages. It receives an array of messages and returns the token count for each message.

The service handles text, image, and file content types. Text is tokenized locally using BPE. Images are tokenized locally using OpenAI's published geometry formulas. Files (PDFs) are tokenized locally by extracting text and estimating per-page image costs. All counting is performed locally with no external service dependencies.

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
      { "type": "input_image", "image_url": "data:image/png;base64,iVBOR..." }
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

## Message Parsing

Each message is a JSON-serialized OpenAI Responses API input item. The service parses each message to identify its content parts and routes each part to the appropriate counting strategy.

A message has one of these shapes:

| Item type | Structure | Content parts |
|-----------|-----------|---------------|
| `message` | `{ "type": "message", "role": "...", "content": [...] }` | Array of typed content parts |
| `function_call` | `{ "type": "function_call", "name": "...", "arguments": "..." }` | Single text blob (the `arguments` string) |
| `function_call_output` | `{ "type": "function_call_output", "output": ... }` | String or array of typed content parts |

Content parts within a `message` or `function_call_output` array are typed:

| Content type | Fields used for counting |
|-------------|------------------------|
| `input_text` | `text` string |
| `output_text` | `text` string |
| `refusal` | `refusal` string |
| `input_image` | `image_url` (URL or data URL) or `file_id` |
| `input_file` | `file_data` (base64) with `filename` |

The token count for a message is the sum of tokens across all its content parts plus per-message overhead tokens (role markers, separators). The overhead is model-specific.

## Counting Strategies

### Text — BPE Tokenization

Text content (`input_text`, `output_text`, `refusal`, `function_call.arguments`, string-valued `function_call_output.output`) is tokenized using BPE (Byte Pair Encoding) with the `o200k_base` encoding. This is the encoding used by the `gpt-4o`, `gpt-4.1`, `gpt-4.5`, and `gpt-5` model families.

The service uses [`tiktoken-go`](https://github.com/pkoukk/tiktoken-go), a Go port of OpenAI's `tiktoken` library. Token counts are exact — they match what OpenAI charges.

### Images — Tile-Based Geometry

Image content (`input_image`) is tokenized locally using OpenAI's published tile-based formula. The service decodes only the image header to read pixel dimensions — it does not decode the full image.

Images arrive as:
- **Data URLs** (`data:image/png;base64,...`): the service decodes the base64 payload header to read dimensions.
- **`image_url`** (HTTPS URL): the service fetches the image headers (HTTP range request or full download) to read dimensions.
- **`file_id`**: not supported for local counting. Counts as zero tokens (the caller is expected to resolve `file_id` to inline data before sending to this service).

#### Tile-Based Formula (gpt-5)

For `detail: "high"` (the default when `detail` is absent):

1. Scale the image to fit within a 2048×2048 square, preserving aspect ratio.
2. Scale so that the shortest side is 768px.
3. Count the number of 512×512 tiles needed to cover the image: `tiles = ceil(width/512) × ceil(height/512)`.
4. `tokens = base_tokens + tile_tokens × tiles`.

For `detail: "low"`:

`tokens = base_tokens`.

Constants for `gpt-5`:

| Parameter | Value |
|-----------|-------|
| `base_tokens` | 70 |
| `tile_tokens` | 140 |

### Files — Local PDF Processing

File content (`input_file`) is tokenized locally by replicating OpenAI's documented PDF processing behavior: extracting text and estimating per-page image cost.

OpenAI processes PDFs by putting both **extracted text** and **an image of each page** into the model's context. The service replicates this:

#### Step 1: Text Extraction

The service parses the PDF and extracts text content from all pages. The extracted text is tokenized using BPE (`o200k_base`), same as any other text content.

The service uses [`pdfcpu`](https://github.com/pdfcpu/pdfcpu), a pure-Go PDF processing library (Apache 2.0 license).

#### Step 2: Per-Page Image Cost

For each page, the service estimates the token cost of the page image that OpenAI renders internally:

1. Read the page dimensions from the PDF MediaBox (in PDF points, 72 points = 1 inch).
2. Convert to pixels at 150 DPI: `width_px = page_width_pt × 150 / 72`, `height_px = page_height_pt × 150 / 72`.
3. Apply the tile-based image formula (same as for `input_image` with `detail: "high"`).

The 150 DPI rendering resolution is an estimate. OpenAI's internal rendering resolution is undocumented. Community reverse-engineering suggests a value between 108 and 150 DPI. The service uses 150 DPI as a conservative upper bound — this slightly overestimates page image tokens, which is safer for summarization budget decisions (overestimating triggers summarization earlier rather than later).

#### Step 3: Total

`file_tokens = text_tokens + sum(page_image_tokens for each page)`

#### Non-PDF Files

Non-PDF text files (`.txt`, `.md`, `.csv`, code files) contain no images. The service extracts the text content from the base64 `file_data` payload and tokenizes it as text.

Binary non-PDF files with unrecognized extensions are not supported. The service returns an error for these.

## Configuration

| Field | Type | Description |
|-------|------|-------------|
| `GRPC_ADDRESS` | string | gRPC listen address (default: `:50051`) |
| `LOG_LEVEL` | string | Log level (default: `info`) |

## Dependencies

| Dependency | Purpose |
|-----------|---------|
| [`tiktoken-go`](https://github.com/pkoukk/tiktoken-go) | BPE tokenization with `o200k_base` encoding |
| [`pdfcpu`](https://github.com/pdfcpu/pdfcpu) | PDF text extraction and page dimension reading |
| Go standard library `image` | Decode image headers for dimensions (PNG, JPEG, GIF, WebP) |
| `net/http` | Fetch remote image dimensions when `image_url` is an HTTPS URL |

No external service dependencies. All token counting is performed locally.

## Token Storage

The agent counts tokens once when it processes a message and persists the count in agent state on disk. The summarization reducer reads stored token counts from agent state rather than re-counting. It sums the per-message counts to decide whether to summarize (`maxTokens` threshold) and uses them for the head/tail split (`keepTokens` budget).
