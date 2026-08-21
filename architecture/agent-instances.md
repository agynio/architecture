# Agent Instances

## Overview

An **agent instance** is a specific, persistent instantiation of an agent [class](agents-service.md). The class defines what an agent *is* (image, model, prompt, tools); the instance is a running (or eligible-to-run) copy of it with its own state and its own inbox.

Every message addressed to an agent lands in exactly one instance's **inbox**. Workloads are scheduled on instances (not on `(agent, thread)` pairs). State is keyed per instance. Threads reference instances as participants; classes are only referenced transiently, at participant-add time, before being resolved to a fresh instance.

This document defines the entity, its inbox, the routing rules that get messages into it, and its lifecycle. Related surfaces:

- [Threads](threads.md) — participants reference instances; class → instance rewrite on add
- [Agents Orchestrator](agents-orchestrator.md) — reconciliation on instances with unacked inbox items
- [Agents Service](agents-service.md) — manages both classes and instances
- [Identity](identity.md) — instance identities and `@nick#suffix` handles
- [`agyn` CLI](agyn-cli.md) — when `--thread` may be omitted on send; class vs. instance in thread commands
- [`agynd`](agynd-cli.md) — inbox fetch/ack protocol replaces per-thread reads

## Entity

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Instance identifier. Also the identity id registered with [Identity](identity.md) with `identity_type = agent_instance` |
| `agent_id` | string (UUID) | Class this instance was spawned from |
| `organization_id` | string (UUID) | Denormalized from the class for authorization scope |
| `label` | string (nullable) | Optional user-chosen handle suffix (see [Handles](#handles)). Unique within the class — `CreateInstance` with a label already taken by a non-terminated instance of the same class returns a conflict error. Unlabeled instances get a system-generated suffix, which cannot collide |
| `default_thread_id` | string (UUID, nullable) | Thread this instance writes to when a message names no target. Set once at creation to the thread the instance was created to serve; not updated when the instance later joins other threads. NULL for instances created outside any thread. See [Outbound](#outbound) |
| `state` | enum | `active` (workload eligible), `paused` (no workload spawns; inbox still accepts writes), `terminated` (soft-deleted; inbox rejects writes) |
| `pause_reason` | string (nullable) | Machine-readable reason set on `active → paused` — e.g., `idle_ttl_exceeded`, `start_failures_exhausted`, `volume_lost`, `runner_deprovisioned`, or `manual`. NULL while `active`. Cleared on `ResumeInstance` |
| `created_at` | timestamp | Creation time |
| `last_activity_at` | timestamp | Set by `agynd` when the workload last did work. Drives idle GC (see [Lifecycle](#lifecycle)) |

An instance is a first-class platform identity. It appears in `MessageRecipient`-equivalent rows, in authorization checks, and in the Identity service's `@mention` resolution.

## Inbox

Every instance owns one inbox: an ordered log of messages addressed to it. The inbox is the single accumulation point for anything the instance needs to process — regardless of source.

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Inbox identifier. One inbox per instance; `inbox.id` is derivable from `instance.id` |
| `instance_id` | string (UUID) | Owning instance |

Inbox items:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Item identifier |
| `inbox_id` | string (UUID) | Parent inbox |
| `source_kind` | enum | `thread` (routed from a thread the instance participates in) or `direct` (written directly by an app via the inbox write API) |
| `thread_id` | string (UUID, nullable) | Set when `source_kind = thread` |
| `message_id` | string (UUID, nullable) | Set when `source_kind = thread` — the underlying [Threads Message](threads.md#message) |
| `sender_id` | string (UUID) | Identity that produced this item |
| `body` | string | Text content |
| `files` | list of string (UUID) | Referenced file IDs |
| `accepted_at` | timestamp | Server-side acceptance time; drives inbox ordering |
| `acked_at` | timestamp (nullable) | NULL = unacknowledged |

**Ordering.** Global FIFO by `accepted_at` across all sources. The instance processes items in acceptance order regardless of which thread they came from. Presentation to the LLM (interleaved vs. grouped) is an agent-implementation concern, not a platform contract.

**Ack semantics.** Items are unacked until the owning workload calls `AckInboxItems`. This ack is about *processing*, and it is the workload's alone. It is unrelated to [thread read state](threads.md#message-delivery): `AckMessages` never touches an inbox item, and acking an inbox item marks nothing read. Unacked items keep the instance in the Orchestrator's reconciliation set (a workload is running or will be started). See [Consumer Sync Protocol](notifications.md#consumer-sync-protocol) for the subscribe/fetch/dedup sequence.

**Ack timing.** The default ack point is **after the turn completes** — the agent has processed the item and posted any responses. A crash before ack means the items are redelivered to the next workload, which may duplicate side effects already performed; this at-least-once behavior is the platform contract. Wrappers that cannot observe turn boundaries in their agent CLI may instead ack once the item is durably accepted into the agent's local state (written to the state volume) — trading redelivery safety for compatibility. Never ack before the item is either processed or durably persisted.

**Write paths.**

- **Thread fan-out** (see [Routing rules](#routing-rules)) — the [Threads](threads.md) service creates one inbox item per participating instance on every `SendMessage`.
- **Direct write** — [apps](apps.md) with the appropriate permission call `WriteInboxItem(instance_id, body, files, ...)` on the Agents service. `source_kind = direct`, `thread_id` and `message_id` are NULL. Useful for connectors/reminders that address the agent without joining a thread.

Reads/acks are always instance-scoped: `GetUnackedInboxItems(instance_id)`, `AckInboxItems(instance_id, item_ids)`. No cross-thread merging inside the agent — the inbox is that merge.

## Participation

Threads reference instances, not classes. A thread [Participant](threads.md#participant) row's `identity_id` is either a user id, an app installation id, or an **agent instance id**. Classes never appear stored on a thread — they are input to participant-add, not output.

### Class-on-add rewrite

`CreateThread` and `AddParticipant` accept either an instance identity or a class identity (i.e., a class `@nickname` or class `identity_id`).

- **Instance provided** — stored as-is. The instance is added to the thread; a new inbox item is created for it on every subsequent `SendMessage`.
- **Class provided** — the Agents service creates a fresh instance of the class (`state = active`), registers its identity, and the stored participant is that instance's id. From the thread's perspective the participant is (and always will be) the instance. The caller learns the instance id on the response.

This makes "spawn a fresh assistant for this conversation" (class-add) and "reuse this specific running assistant" (instance-add) explicit at the participant level, with a uniform downstream storage shape.

### Delegation across threads

Preserving context across a sub-thread is achieved by adding **the same instance** to both threads. When agent instance A (running on thread X) creates thread Y and adds itself as an instance (not as its class), messages on Y for A route to A's inbox — the same inbox A processes for X. No separate workload on Y, no fresh context.

### Passive participation removed

The [`passive` participant flag](threads.md) is removed. Every participant is a consumer of its inbox by default. The original passive use case (A subscribes to Y's responses without spawning a new workload on Y) is served by explicit instance participation: A adds `@a#N` (its running instance), not `@a` (the class).

## Routing rules

On `SendMessage(thread_id, sender_id, body, files)`:

1. Look up thread participants (excluding `sender_id`).
2. For each participant, resolve identity type via [Identity](identity.md):
   - `user`, `app` — notification to `thread_participant:{participant_id}`.
   - `agent_instance` — create an inbox item on the participant instance's inbox with `source_kind = thread`, `thread_id`, `message_id`, `sender_id`, `body`, `files`, `accepted_at = server time`. Notification to `instance_inbox:{instance_id}`.

Every non-sender participant also gets a [`MessageRecipient`](threads.md#messagerecipient) row, whatever its type. That row is [read state](threads.md#message-delivery), not delivery — it records what an identity has consumed on the thread and gates nothing. The delivery paths above are the part that is mutually exclusive: an instance is woken through its inbox and never through `thread_participant:`, and no user or app ever gets an inbox item.

Cross-thread ordering is not guaranteed globally, only within a single inbox. A `SendMessage` on thread X at wall-clock time T and a `SendMessage` on thread Y at time T' > T are ordered `T` before `T'` inside any inbox that receives items from both.

## Outbound

[Routing rules](#routing-rules) cover what arrives in the inbox. This section covers where the instance's own messages go.

**Explicit targeting is the rule.** Every `Threads.SendMessage` from an instance names its target thread. An instance participating in several threads — the normal case once agents delegate — decides per message which one it is writing to, because *the thread an item arrived on is not the thread the answer belongs on*. In a chain A → B → C, C's reply reaches B on thread BC, but B's answer to A belongs on thread AB. No rule derived from the triggering item gets this right; only the agent knows.

### Default thread

`default_thread_id` is the instance's fallback target — the thread it was created to serve.

- **Set once, at creation.** On the [class-on-add](#class-on-add-rewrite) path the joining thread is assigned automatically when the class sets [`default_thread = origin`](resource-definitions.md#agent), which is the default; under `none` the field is left NULL. On `agyn agents instantiate` it comes from the optional `--default-thread` whatever the class policy says — an explicit choice is not overridden.
- **Not moved by joining.** Adding the instance to further threads leaves it alone. It records origin ("who I owe an answer to"), not recency ("where I last heard something"). Changing it is an explicit operation — [`SetInstanceDefaultThread`](agents-service.md#agent-instance-api), requiring `can_manage`.
- **Resolved server-side.** `Threads.SendMessage` with no `thread_id` from an `agent_instance` caller delivers to that instance's `default_thread_id`. This is what makes `--thread` optional on [`agyn threads send`](agyn-cli.md#thread-commands) inside an agent workload. Callers that are not agent instances, and instances with a NULL default, get an error — the fallback is a property of the caller's platform identity, never of anything in the container's environment.

Because delegation creates sub-threads downward, origin semantics compose: B's default is AB, C's default is BC, and an untargeted message from each travels back up the chain. Writing *sideways* — onto a sub-thread the instance itself created — always requires an explicit `thread_id`.

### Final turn message

Agent CLIs end a turn with plain assistant text that carries no thread target. What becomes of that text is controlled by [`final_message`](resource-definitions.md#agent) on the **class**, default `discard`:

| Value | Behavior |
|-------|----------|
| `discard` | The final text is dropped. The agent's output reaches threads only through explicit sends. Default, because an agent that manages its own sends would otherwise post everything twice |
| `default_thread` | [`agynd`](agynd-cli.md) posts the turn's final text to the instance's `default_thread_id`, after the turn completes and before [ack](#inbox) |

Under `default_thread` the post is unconditional — `agynd` does not attempt to detect whether the agent already sent something itself. Which value fits is a statement about how the agent is written, which is why it lives on the class rather than the instance.

`default_thread` with a NULL `default_thread_id`, or an empty final text, posts nothing and logs. Nothing is guessed.

This is the path that carries a plain conversational agent's reply back to a user in [Chat](chat.md) — one instance, one thread, no thread-id bookkeeping in the prompt. It also gives `source_kind = direct` items somewhere to be answered: an instance woken by an app's direct write has no thread on the item, but it does have a default.

## Configuration

Instances read **live class configuration** — there is no per-instance config snapshot. A workload picks up the class's current definition (image, model, prompt, sub-resources) at start; the [configuration-driven fast retry](agents-orchestrator.md#start-decision) already assumes this. Consequently, deleting a class is **blocked while it has non-terminated instances** — callers must terminate (or let expire) all instances first. Per-instance configuration overrides are out of scope for the initial model.

## Lifecycle

### Creation

- **Lazy** — a fresh instance is created by `CreateThread` / `AddParticipant` when a class is provided. This is the primary path. Threads supplies the thread being joined as [`context.thread_id`](agents-service.md#creation-context) on `CreateInstance`; the Agents service applies the class's [`default_thread`](resource-definitions.md#agent) policy to decide whether it becomes the instance's [`default_thread_id`](#default-thread).
- **Explicit** — `agyn agents instantiate @class [--label LABEL] [--default-thread REF]` creates an instance without joining a thread. Useful when an app wants to address an instance directly before any conversation exists. `default_thread_id` is NULL unless `--default-thread` is given.

Instance creation is never performed by the [Agents Orchestrator](agents-orchestrator.md) — it reconciles instances that already exist and have unacked inbox items, and starts workloads for them. Every instance is created by `Agents.CreateInstance`, on one of the two paths above.

The class must satisfy the agent [availability](agents-service.md#availability) check for the caller (`can_initiate`). Instance creation is authorized by the same rule as adding the class to a thread. Creating a labeled instance whose `label` collides with a non-terminated instance of the same class returns a conflict error.

### Idle GC and pausing

Instances persist beyond individual workload lifecycles. State on disk (see [Agent State](agent/state.md)) survives workload restarts. A workload is only running when there's work to do; the instance stays around.

An instance transitions `active → paused` (with a `pause_reason`) when:

- Idle for more than the class's [`instance_idle_ttl`](../architecture/resource-definitions.md#agent) (default 30 days) without new inbox items (`idle_ttl_exceeded`).
- The [Orchestrator](agents-orchestrator.md) exhausts start retries (`start_failures_exhausted`), loses the instance's volume (`volume_lost`), or the hosting runner is deprovisioned (`runner_deprovisioned`).
- Explicit `PauseInstance(instance_id)` by an authorized caller (`manual`).

`paused` means: no workloads spawn, but the inbox continues to accept writes — no data is lost while the owner investigates. `ResumeInstance` clears `pause_reason` and flips the instance back to `active`; pending inbox items are picked up on the next reconciliation tick.

An instance transitions to `terminated` on explicit `DeleteInstance`. Terminated is soft-delete — the inbox rejects new writes, existing state is retained for audit until a hard retention cutoff.

### State and workload

- **Storage** is per-instance. Every persistent [Volume](resource-definitions.md#volume) the instance's [environment](resource-definitions.md#environment) declares is provisioned once per instance — records in [Runners](runners.md) are keyed by `agent_instance_id`. Workload starts mount them; workload stops leave them in place. An environment declaring no persistent volume gives its instances no state that outlives a workload.
- **Workload identity** is the instance identity. The [Orchestrator](agents-orchestrator.md) starts a workload with `AGENT_INSTANCE_ID = instance.id` in the environment; `agynd` uses this to fetch its inbox and to identify itself in platform calls. (In [Runners](runners.md) records the field is `agent_instance_id` — distinct from the runner-local `instance_id`, which is the Pod/PVC name.)
- One workload at a time per instance. A `starting`/`running`/`stopping` workload precludes another for the same instance.

## Handles

Instance handles use the form `@nick#suffix`:

- `@nick` — the class's nickname (unchanged; still stored on the class in the [Identity](identity.md) nickname index).
- `#suffix` — instance-scoped. Either system-generated (opaque, e.g., `#7a2f`) or the instance's optional `label` field (user-chosen).

Handle resolution: `ResolveHandle("@bob#research", org_id)` returns the instance identity id; `ResolveHandle("@bob", org_id)` returns the class identity id. The `#` split is a first-class part of the resolver.

Uniqueness: `UNIQUE(agent_id, label)` for labeled instances within a class. System-generated suffixes are UUID-derived and cannot collide.

## Authorization

Instance authorization derives from the class. Adding an instance to a thread requires `can_initiate` on the class (same as adding the class itself). Lifecycle operations (`Pause`/`Resume`/`Delete`) require `can_manage`, derived from the class's `can_delete`. Direct inbox writes require `can_write_inbox` — held via direct grant or the app-level [`inbox:write` permission](apps.md#permissions). Inbox reads and acks are self-only (identity equality), like `TouchWorkload`.

The `agent_instance` identity itself holds `member` on its organization (via its class) — enough for `agynd` to make platform API calls in the same way today's agent workload identities do. Full model: [Authorization — agent_instance type](authz.md#agent_instance).

## Relationship to existing concepts

| Concept | Change |
|---------|--------|
| Agent (class) | Unchanged — still the configuration entity in [Agents Service](agents-service.md) |
| Agent workload | Now scoped to an instance (was: `(agent, thread)`) |
| Thread participant | References an instance (was: an agent identity) |
| `MessageRecipient` | Only created for `user`/`app` participants; agent-instance participants get inbox items instead |
| `passive` participant | Removed |
| Agent state storage | Keyed by instance (was: keyed by `(agent, thread)`) |
| `THREAD_ID` env var | Removed; replaced by `AGENT_INSTANCE_ID` |
| Implicit reply thread | Was the workload's `THREAD_ID`. Now [`default_thread_id`](#outbound) on the instance, resolved server-side from the caller identity — and opt-in for the final turn message |
| `thread_participant:{id}` notification room (for agents) | Replaced by `instance_inbox:{id}` for agent instances; users and apps still use `thread_participant` |
