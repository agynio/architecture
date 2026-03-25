# Memory

## Overview

Memory is a platform service that allows an agent to retain past information and use it to influence future behavior. Memories are stored as `(key, content)` pairs scoped per-agent. Instead of exact matching, a learned activation function evaluates the current context against all stored memory keys and assigns a relevance probability to each memory. The most relevant memories are injected into the agent's context via [hooks](agent/overview.md). The system continuously improves through agent-provided feedback scores, which train the activation function over time.

Memory is a **data plane** service — activation runs on the hot path during agent inference.

## Data Model

### Memory

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Unique memory identifier |
| `agent_id` | string (UUID) | Owning agent. All memories are scoped per-agent |
| `key` | Key | Composite retrieval key (see below) |
| `content` | Content | Memory payload (see below) |
| `created_at` | timestamp | Creation time |

### Key

A composite object describing the context in which the memory should be activated.

| Field | Type | Description |
|-------|------|-------------|
| `repo` | string (nullable) | Repository or domain the memory belongs to. `null` if the memory is a general rule not tied to a specific repo |
| `context` | string | Short natural-language description of when this memory should be activated (the retrieval trigger condition) |

The serialization format for keys (JSON, YAML, or other) is an [open question](#key-serialization-format).

### Content

| Field | Type | Description |
|-------|------|-------------|
| `text` | string | The memory payload — what the agent should remember or how it should behave |
| `priority` | enum | Importance level: `low`, `medium`, `high`. Reflects how strongly the memory should influence agent behavior. A hard constraint (e.g., "never use library X") carries `high` priority; a soft stylistic preference carries `low` |

## Memory Creation

The agent creates memories via a dedicated tool. The tool accepts a key and content and persists the memory in the Memory service. There is no manual creation interface — memories are created exclusively by agents during execution.

### Cold Start

When creating a memory, the agent should also provide positive and negative examples of keys — contexts where the memory should and should not be activated. These examples serve as initial training data for the activation function, allowing it to make reasonable retrieval decisions before real feedback scores accumulate.

## Activation Function

The activation function is a trained model that evaluates the current context against all stored memory keys for an agent and returns a relevance probability for each memory.

| Aspect | Details |
|--------|---------|
| **Input** | A text key (the current context) |
| **Output** | Activation probability `[0, 1]` for each stored memory |
| **Selection** | Top-K + threshold filtering: only memories that exceed the activation probability threshold **and** rank in the top-K are selected for injection. This prevents memories from being surfaced on every call, reducing noise and context bloat |
| **Training** | Continuously fine-tuned using agent-provided feedback scores (see [Scoring](#scoring)) |

### Architecture

The activation model uses a **pretrained encoder** (taken from a lightweight LLM suitable for running on a local machine) with a **lightweight trainable head per agent**. The shared encoder provides strong text representations out of the box; only the per-agent head is trained on feedback data, keeping compute and storage requirements low.

Each agent has its own activation model (shared encoder + agent-specific head). How to store per-agent models is an [open question](#per-agent-model-storage). Which encoder to use is an [open question](#encoder-selection).

## Injection via Hooks

At inference time, hooks automatically run the activation function and inject the selected memories into the tool message. The agent receives relevant memories without making explicit retrieval calls.

### Deduplication

The system tracks which memories have already been injected in the current session to avoid re-injection and context window bloat. How long an injected memory persists in context — and what defines a "session" — is an [open question](#memory-lifetime-and-session-boundary).

## Scoring

After a memory has been used, the agent scores its usefulness on a `[0, 1]` scale. These scores serve as training signal to continuously fine-tune the activation function, closing the feedback loop.

### Timing

Scoring does not happen immediately after injection — the agent needs time to actually use (or ignore) the memory. The natural trigger is at the end of the memory's lifetime: just before deletion or at a logical session boundary.

### Label Storage

Feedback labels (memory ID, context key, score) must be persisted for training. How and where to store labels is an [open question](#label-storage).

## Memory Lifecycle

There are no explicit update or delete operations. The lifecycle is entirely score-driven: a memory that consistently scores near `0` falls below the activation threshold and effectively ceases to be injected, becoming dormant. This keeps the mechanism simple and self-regulating.

```
Creation → Activation → Injection → Usage → Scoring → Retraining
    ↑                                                      |
    |              (feedback loop)                          |
    +-------------------------------------------------------+
```

## Open Questions

### Key Serialization Format

The memory key is a composite object with `repo` and `context` fields. The wire format for serializing keys (JSON, YAML, or another format) has not been decided.

### Memory Lifetime and Session Boundary

How long an injected memory persists in context is an open research question. Three candidate approaches:

1. **Message-counter + sticky mode.** Each memory carries a TTL counter (e.g., 5 LLM calls). On every turn, active memories are concatenated into a `<memory_block>` appended to the latest tool message. The counter decrements each call; at zero the memory is dropped. *Upside:* memories are always attached to the most recent context, keeping them fresh. *Downside:* memory is decoupled from the specific tool call that triggered it.

2. **Pin to originating tool call, no expiry.** Memory is injected once into the output of the tool call that activated it and stays there permanently. Per-thread injection state is tracked to avoid re-injection. *Upside:* full traceability — you know exactly which tool call surfaced which memory. *Downside:* stale memories accumulate; context grows unboundedly over long threads.

3. **Agent-controlled expiry.** Same TTL counter as option 1, but before deletion the agent is asked whether the memory should be retained. The agent can extend its lifetime or let it expire. *Upside:* more adaptive. *Downside:* adds LLM calls and complexity; agent judgment may be inconsistent.

### Label Storage

Feedback labels (memory ID, context key, score) are produced by the agent after memory usage. How and where to store these labels has not been decided (e.g., in the Memory service database, a separate training data store, or alongside the model artifacts).

### Per-Agent Model Storage

Each agent has its own activation model (shared encoder + agent-specific trainable head). How to store and serve per-agent model weights has not been decided (e.g., object storage, model registry, or alongside the Memory service database).

### Encoder Selection

The activation model architecture uses a pretrained encoder from a lightweight LLM with a trainable head per agent. Which specific encoder to use has not been decided.

### Training Update Strategy

How the activation model is retrained as new feedback labels arrive. Two candidate approaches:

1. **Mini-batch.** Accumulate labels until a batch of sufficient size is collected, then run a training step. *Upside:* stable gradient updates, standard training practice. *Downside:* delay between feedback and model improvement.

2. **Per-step with small learning rate.** Train on each new label immediately with a very small learning rate. *Upside:* fastest adaptation. *Downside:* potential instability from noisy single-sample updates.
