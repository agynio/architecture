# Token Counting

## Overview

A Go package within [`agn`](agn-cli.md) that counts tokens for LLM messages. It is invoked in-process during agent execution — there is no separate service, no network call, no extra runtime dependency.

The package accepts an array of messages and returns the token count for each. It handles text, image, and file content types. Text is tokenized using BPE. Images are counted using OpenAI's published tile-based geometry. Files (PDFs) are counted by extracting text and estimating per-page image costs. All counting is performed in-process.

## Motivation

- **litellm's `input_tokens`** was evaluated and does not provide reliable pre-call token counts.
- Summarization needs to know token counts **before** the LLM call to decide whether to summarize.
- Media files (images, documents) have token costs that cannot be estimated from byte size or string length.
- Embedding the counter inside `agn` keeps the agent's data-plane dependency surface minimal — the only required dependency is the LLM endpoint (with optional tracing). A standalone token-counting service would add network hops on the hot path and an extra component to operate, for code and tables (~2MB BPE merges, a Go-native PDF parser) that fit comfortably in the binary.

## Interface

The package exposes a single function.

### CountTokens

**Input:**

| Field | Type | Description |
|-------|------|-------------|
| `model` | enum | Target model. Supported values: `gpt-5` |
| `messages` | list of Message | Messages to count tokens for. Each message is a serialized OpenAI Responses API input/output item |

Currently only OpenAI Responses API format is supported. The package is designed to be extended with other formats in the future.

**Output:**

| Field | Type | Description |
|-------|------|-------------|
| `tokens` | list of integer | Token count per message, in the same order as the input |

The output array has the same length as the input `messages` array. Each element is the token count for the corresponding message.

### Example

**Input:**
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

**Output:**
```json
{
  "tokens": [12, 4, 1028]
}
```

## Message Parsing

Each message is a JSON-serialized OpenAI Responses API input item. The package parses each message to identify its content parts and routes each part to the appropriate counting strategy.

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

The package uses [`tiktoken-go`](https://github.com/pkoukk/tiktoken-go), a Go port of OpenAI's `tiktoken` library. Token counts are exact — they match what OpenAI charges.

### Images — Tile-Based Geometry

Image content (`input_image`) is tokenized using OpenAI's published tile-based formula. The package decodes only the image header to read pixel dimensions — it does not decode the full image.

Images arrive as:
- **Data URLs** (`data:image/png;base64,...`): the package decodes the base64 payload header to read dimensions.
- **`image_url`** (HTTPS URL): the package fetches the image headers (HTTP range request or full download) to read dimensions.
- **`file_id`**: not supported for local counting. Counts as zero tokens (the caller is expected to resolve `file_id` to inline data before counting).

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

File content (`input_file`) is tokenized by replicating OpenAI's documented PDF processing behavior: extracting text and estimating per-page image cost.

OpenAI processes PDFs by putting both **extracted text** and **an image of each page** into the model's context. The package replicates this:

#### Step 1: Text Extraction

The package parses the PDF and extracts text content from all pages. The extracted text is tokenized using BPE (`o200k_base`), same as any other text content.

The package uses [`pdfcpu`](https://github.com/pdfcpu/pdfcpu), a pure-Go PDF processing library (Apache 2.0 license).

#### Step 2: Per-Page Image Cost

For each page, the package estimates the token cost of the page image that OpenAI renders internally:

1. Read the page dimensions from the PDF MediaBox (in PDF points, 72 points = 1 inch).
2. Convert to pixels at 150 DPI: `width_px = page_width_pt × 150 / 72`, `height_px = page_height_pt × 150 / 72`.
3. Apply the tile-based image formula (same as for `input_image` with `detail: "high"`).

The 150 DPI rendering resolution is an estimate. OpenAI's internal rendering resolution is undocumented. Community reverse-engineering suggests a value between 108 and 150 DPI. The package uses 150 DPI as a conservative upper bound — this slightly overestimates page image tokens, which is safer for summarization budget decisions (overestimating triggers summarization earlier rather than later).

#### Step 3: Total

`file_tokens = text_tokens + sum(page_image_tokens for each page)`

#### Non-PDF Files

Non-PDF text files (`.txt`, `.md`, `.csv`, code files) contain no images. The package extracts the text content from the base64 `file_data` payload and tokenizes it as text.

Binary non-PDF files with unrecognized extensions are not supported. The package returns an error for these.

## Initialization

Encoding tables (`o200k_base` BPE merges) are loaded once at `agn` startup. There is no runtime configuration — the package is invoked directly by the summarization reducer.

## Dependencies

| Dependency | Purpose |
|-----------|---------|
| [`tiktoken-go`](https://github.com/pkoukk/tiktoken-go) | BPE tokenization with `o200k_base` encoding |
| [`pdfcpu`](https://github.com/pdfcpu/pdfcpu) | PDF text extraction and page dimension reading |
| Go standard library `image` | Decode image headers for dimensions (PNG, JPEG, GIF, WebP) |
| `net/http` | Fetch remote image dimensions when `image_url` is an HTTPS URL |

No external service dependencies. All token counting is performed in-process within `agn`.

## Token Storage

`agn` counts tokens once when it processes a message and persists the count in agent state on disk. The summarization reducer reads stored token counts from agent state rather than re-counting. It sums the per-message counts to decide whether to summarize (`maxTokens` threshold) and uses them for the head/tail split (`keepTokens` budget).
