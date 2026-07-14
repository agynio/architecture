# Agent Instances and Inboxes

## Target

- [Architecture: Agent Instances](../architecture/agent-instances.md)
- [Architecture: Threads — Interface](../architecture/threads.md#interface)
- [Architecture: Threads — Participant](../architecture/threads.md#participant)
- [Architecture: Threads — Message Delivery](../architecture/threads.md#message-delivery)
- [Architecture: Threads — Class-on-Add Rewrite](../architecture/threads.md#class-on-add-rewrite)
- [Architecture: Threads — Agent Availability Check](../architecture/threads.md#agent-availability-check)
- [Architecture: Threads — Runtime Associations](../architecture/threads.md#runtime-associations)
- [Architecture: Agents Orchestrator](../architecture/agents-orchestrator.md)
- [Architecture: Agents Service — Resources](../architecture/agents-service.md#resources)
- [Architecture: Agents Service — Agent Instance API](../architecture/agents-service.md#agent-instance-api)
- [Architecture: Agents Service — Inbox API](../architecture/agents-service.md#inbox-api)
- [Architecture: Agents Service — Notifications](../architecture/agents-service.md#notifications)
- [Architecture: Identity — Identity Model](../architecture/identity.md#identity-model)
- [Architecture: Identity — Nickname Index](../architecture/identity.md#nickname-index)
- [Architecture: Notifications — Room Naming Convention](../architecture/notifications.md#room-naming-convention)
- [Architecture: Notifications — Self-Subscription Sentinel](../architecture/notifications.md#self-subscription-sentinel)
- [Architecture: Runners — Workload Resource](../architecture/runners.md#workload-resource)
- [Architecture: Runners — Volume Resource](../architecture/runners.md#volume-resource)
- [Architecture: Runners — Agent Instance Associations](../architecture/runners.md#agent-instance-associations)
- [Architecture: Chat — Activity Status](../architecture/chat.md#activity-status)
- [Architecture: Authorization — agent_instance type](../architecture/authz.md#agent_instance)
- [Architecture: Authorization — App Installation Permissions](../architecture/authz.md#app-installation-permissions)
- [Architecture: Apps — Permissions](../architecture/apps.md#permissions)
- [Architecture: Resource Definitions — Agent](../architecture/resource-definitions.md#agent)
- [Architecture: Tracing — agynd Tracing Proxy](../architecture/tracing.md#agynd-tracing-proxy)
- [Architecture: Agent Init — Environment Variable Contract](../architecture/agent-init.md#environment-variable-contract)
- [Architecture: agyn CLI — Thread Commands](../architecture/agyn-cli.md#thread-commands)
- [Architecture: agynd CLI](../architecture/agynd-cli.md)
- [Architecture: Agent — Overview](../architecture/agent/overview.md)
- [Architecture: Agent State](../architecture/agent/state.md)
- [Architecture: Gateway — Exposed Services](../architecture/gateway.md)
- [Architecture: Console](../architecture/console.md)

## Delta

### Agents Service — Instances and Inbox

- `agent_instances` table does not exist. `CreateInstance`, `GetInstance`, `ListInstances`, `PauseInstance`, `ResumeInstance`, `DeleteInstance` do not exist. `pause_reason` semantics (set on pause, cleared on resume) are unimplemented.
- Instance identity registration with the Identity service does not exist (no `identity_type=agent_instance` writer).
- `inbox_items` table does not exist. `WriteInboxItem`, `GetUnackedInboxItems`, `AckInboxItems`, `GetUnackedInboxCount` (with optional `thread_id` filter), and the internal `FanoutInboxItem` RPC do not exist.
- Instance idle GC (background loop that transitions `active → paused` past `instance_idle_ttl` with `pause_reason=idle_ttl_exceeded`) is not implemented.
- `DeleteAgent` has no precondition on non-terminated instances (nothing to check yet — the entity doesn't exist).
- Label conflict detection on `CreateInstance` (unique non-terminated `label` per class) does not exist.
- `instance.updated` and inbox-scoped `message.created` events on `instance_inbox:{id}` are not emitted.
- `ResolveAgentIdentity` resolves an agent (class) identity; it does not resolve instance identities or return `agent_instance_id`.

### Threads Service

- `Participant` stores a generic identity id with no class/instance distinction. The class → fresh-instance rewrite on `CreateThread` / `AddParticipant` does not exist.
- `SendMessage` creates `MessageRecipient` rows for every non-sender participant regardless of identity type. The per-type delivery branch (users/apps → `MessageRecipient`, agent instances → inbox item via `FanoutInboxItem`) does not exist.
- `SendMessage` publishes only to `thread_participant:{id}` rooms; no `instance_inbox:{id}` publication.
- The Agent Availability Check handles `identity_type == agent` only; it does not resolve `agent_instance → class` for the `can_initiate` check.
- `AddParticipant` resolves `@nickname` but not `@nickname#suffix` instance handles.

### Identity Service

- The nickname index does not exist at all: `SetNickname`, `RemoveNickname`, `ResolveNickname` methods and `org_nicknames` storage are unimplemented. (Carried over from the superseded [2026-04-12 change](#notes).)
- `identity_type` enum does not include `agent_instance`.
- `instance_suffix` column and `@nickname#suffix` parsing in `ResolveNickname` do not exist.
- User profile update does not support `nickname` (per-org handle); agents do not register nicknames on create/update; app installations do not register a default nickname on install. (Carried over from 2026-04-12.)

### Notifications Service

- `instance_inbox:{id}` room type has no producer or consumer.
- The `:me` sentinel rewrite covers only `thread_participant:`; `instance_inbox:me` (with the `identity_type == agent_instance` restriction) is not implemented.
- Room authorization has no rule for `instance_inbox:{id}`.

### Agents Orchestrator

- Reconciliation desired-state query reads unacked messages from Threads per `(thread_id, agent_id)`. It does not call `Agents.ListInstances(state=active, has_unacked=true)`.
- Start Decision, Runner Selection, and volume TTL logic operate on `(thread_id, agent_id)` via `ListWorkloadsByThread` / `ListVolumesByThread`. They need to operate on `agent_instance_id` via `ListWorkloadsByAgentInstance` / `ListVolumesByAgentInstance`.
- Terminal failure handling calls `Threads.DegradeThread`. It needs to call `Agents.PauseInstance` with a `pause_reason` (`start_failures_exhausted`, `volume_lost`, `runner_deprovisioned`) — threads are no longer degraded by instance-scoped failures.
- Workload spec assembly injects `THREAD_ID`. It needs to inject `AGENT_INSTANCE_ID` instead (alongside the existing `AGENT_ID` for class config).
- Workload/volume records are created with `thread_id`; they need `agent_instance_id`.

### Runners Service

- Workload and Volume records have `thread_id` but no `agent_instance_id`. The naming collision with the runner-local `instance_id` (Pod/PVC name) must be resolved by adding the distinct `agent_instance_id` field — the runner-local field keeps its name.
- `ListWorkloadsByThread` / `ListVolumesByThread` exist; `ListWorkloadsByAgentInstance` / `ListVolumesByAgentInstance` do not. Gateway and Console wiring follow the rename.
- `TouchWorkload` authorizes against `workload.agent_identity_id`; it needs to authorize against `workload.agent_instance_id` (the caller's identity is the instance).

### Chat Service

- `activity_status` derivation reads workloads per `(thread, agent)` pair and skips passive participants. It needs to derive per agent-instance participant via `ListWorkloadsByAgentInstance`, including the `failed + instance paused → finished` row.
- `CreateChat` with an agent class does not surface the resolved instance handle to the UI.

### Authorization

- `agent_instance` OpenFGA type (relations `class`, `org`, `can_initiate`, `can_manage`, `can_write_inbox`) does not exist.
- `inbox_write` relation on the `organization` type and the `inbox:write` app-permission tuple write do not exist.
- Tuple lifecycle for instance create/delete is unimplemented.

### Apps Service

- `inbox:write` is not in the app permission vocabulary; installation does not write the `inbox_write` tuple.

### Resource Definitions

- `instance_idle_ttl` field does not exist on the Agent resource.

### Tracing

- The `agynd` tracing proxy injects `agyn.thread.id` statically from `THREAD_ID`. It needs to inject `agyn.agent_instance.id` statically (from `AGENT_INSTANCE_ID`) and `agyn.thread.id` / `agyn.thread.message.id` per-turn from the current inbox items.
- The Tracing service does not verify `agyn.agent_instance.id` against the caller identity, and `ResolveAgentIdentity` consumers do not handle the instance → class resolution.

### agynd

- Subscribes to `thread_participant:me`; needs `instance_inbox:me`.
- Reads via `Threads.GetUnackedMessages(thread_id)`; needs `Agents.GetUnackedInboxItems(AGENT_INSTANCE_ID)`. Acks via `Threads.AckMessages`; needs `Agents.AckInboxItems`.
- No `AGENT_INSTANCE_ID` env handling.
- Message formatting lacks the per-item `thread:` / `from:` (or `source: direct`) header.
- Ack timing contract (after turn completion; state-durability fallback for SDKs without turn boundaries) is unimplemented.

### agyn CLI

- The `threads` command group does not exist at all: `create`, `send`, `read`, `add`, `list`, plus local ref state (`~/.agyn/threads.json`), `--wait`, `--unread` / `--tail` / `--limit`, multi-thread read, and `--json` / `--yaml` output flags are all unimplemented. (Carried over from 2026-04-12 — with these deltas superseding it: no `--passive` flag, no `THREAD_ID` fallback, `--thread` mandatory on `send`.)
- `agyn threads reply --to-message MSG_ID` does not exist.
- `agyn agents instantiate @class [--label LABEL]` does not exist.
- Handle resolution does not parse `@nickname#suffix`.
- Self-participation as instance (agent joining threads it creates using its `AGENT_INSTANCE_ID` identity) does not exist.

### Agent Implementation (`agn`)

- Receives messages as untagged plain text; needs to accept the thread-source header and track `thread_id` per inbound/outbound message in state.
- Outbound send tooling has no required `thread_id` parameter (ripples into `agn-sdk-go`).
- State layout is not keyed per instance.

### Agent Init

- Env contract provides `THREAD_ID`; needs `AGENT_INSTANCE_ID`.

## Acceptance Signal

- Creating a thread with `agyn threads create --thread research --add @bob --message "..."` returns the resolved participant handle (e.g., `@bob#7a2f`) and creates a fresh `agent_instance`.
- Adding an existing instance to a second thread does not create a new instance; a `SendMessage` on either thread produces exactly one inbox item on that instance's inbox, and inbox items from both threads interleave in acceptance order.
- An agent instance A running with items from thread X can create sub-thread Y (adding itself as instance, `@c` as class) and receive `@c#N`'s reply as an inbox item processed by A's already-running workload — no second workload for A, reply tagged with Y's thread id.
- `agyn threads send` without `--thread` fails with an error even inside an agent workload; no environment variable substitutes for it.
- An app installation that declared `inbox:write` can call `WriteInboxItem(instance_id, ...)`; the instance's workload wakes and the item arrives tagged `source: direct`. An app without the permission is rejected.
- `CreateInstance` with a `label` already used by a non-terminated instance of the same class returns a conflict error; unlabeled creation always succeeds with a generated suffix.
- `DeleteAgent` on a class with non-terminated instances is rejected; after terminating them it succeeds.
- Start-failure exhaustion, volume loss, and runner deprovisioning pause the instance with the corresponding `pause_reason`; the thread stays writable and inbox items accumulate. `ResumeInstance` clears the reason and the Orchestrator picks up pending items on the next tick.
- Killing a workload and starting a new one for the same instance mounts the same state volume; agent state survives.
- Workload records in Runners carry both the runner-local `instance_id` (Pod name) and `agent_instance_id`; Console Thread Detail resolves workloads via the thread's instance participants.
- Spans exported from an agent workload carry `agyn.agent_instance.id` matching the workload identity, and `agyn.thread.id` matching the turn's source thread; a forged `agyn.agent_instance.id` is rejected.
- Chat shows `running` for every chat whose instance participant has a `running`/`processing` workload, and `finished` (not `pending`) when the instance is paused after retry exhaustion.

## Notes

- **Supersedes and absorbs [2026-04-12 Agent-to-Agent Communication](2026-04-12-agent-to-agent-communication.md)** (file deleted; git history preserves it). Still-valid deltas from it — the nickname index, nickname registration across users/agents/apps, and the `agyn threads` command group — are carried in this file. Its passive-participant model (`--passive` flag, `Participant.passive` column, orchestrator passive skip) and `THREAD_ID`-scoped message processing are dropped entirely, replaced by instance participation and the inbox.
- **Naming.** The runner-local `instance_id` (Pod/PVC name) on Runners records keeps its name; the platform-level field is `agent_instance_id` everywhere (records, RPCs, env var `AGENT_INSTANCE_ID`, tracing attribute `agyn.agent_instance.id`). Renaming the runner-local field was rejected to avoid churn in runner/k8s docs and code.
- **Live class config, no snapshots.** Instances read the class definition at workload start; class deletion is blocked while non-terminated instances exist. Per-instance config overrides are explicitly out of scope.
- **Ack timing.** Default is ack-after-turn-completion (at-least-once, redelivery on crash). SDKs that cannot observe turn boundaries may ack after durable write to the state volume. Never ack before one of the two.
- **Thread degradation** remains in Threads (status, `DegradeThread`, consumer handling like Telegram rotation) but no current platform flow produces it — instance-scoped failures pause the instance instead. Removing the status entirely is a possible future cleanup once confirmed nothing else needs it.
- **`agn-sdk-go` coordination.** The required `thread_id` on outbound sends is an SDK API change; coordinate with the `agn` repo.
- **Migration.** No production data to preserve; pre-production threads/agents are recreated in the new model.
- **Console / UI.** Instance surfaces (list, detail, inbox activity, pause/resume controls) need their own product/console delta. Track separately.
