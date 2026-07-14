# Agent Instances and Inboxes

## Target

- [Architecture: Agent Instances](../architecture/agent-instances.md)
- [Architecture: Threads — Interface](../architecture/threads.md#interface)
- [Architecture: Threads — Participant](../architecture/threads.md#participant)
- [Architecture: Threads — MessageRecipient](../architecture/threads.md#messagerecipient)
- [Architecture: Threads — Message Delivery](../architecture/threads.md#message-delivery)
- [Architecture: Threads — Class-on-Add Rewrite](../architecture/threads.md#class-on-add-rewrite)
- [Architecture: Threads — Agent Availability Check](../architecture/threads.md#agent-availability-check)
- [Architecture: Threads — Notification Publishing](../architecture/threads.md#notification-publishing)
- [Architecture: Agents Orchestrator — Overview](../architecture/agents-orchestrator.md#overview)
- [Architecture: Agents Orchestrator — Reconciliation](../architecture/agents-orchestrator.md#reconciliation)
- [Architecture: Agents Orchestrator — Start Decision](../architecture/agents-orchestrator.md#start-decision)
- [Architecture: Agents Orchestrator — Runner Selection](../architecture/agents-orchestrator.md#runner-selection)
- [Architecture: Agents Orchestrator — Workload Spec Assembly](../architecture/agents-orchestrator.md#workload-spec-assembly)
- [Architecture: Agents Service — Resources](../architecture/agents-service.md#resources)
- [Architecture: Agents Service — Agent Instance API](../architecture/agents-service.md#agent-instance-api)
- [Architecture: Agents Service — Inbox API](../architecture/agents-service.md#inbox-api)
- [Architecture: Agents Service — Notifications](../architecture/agents-service.md#notifications)
- [Architecture: Identity — Identity Model](../architecture/identity.md#identity-model)
- [Architecture: Identity — Nickname Index](../architecture/identity.md#nickname-index)
- [Architecture: Identity — Registration](../architecture/identity.md#registration)
- [Architecture: Notifications — Room Naming Convention](../architecture/notifications.md#room-naming-convention)
- [Architecture: Notifications — Self-Subscription Sentinel](../architecture/notifications.md#self-subscription-sentinel)
- [Architecture: Notifications — Authorization](../architecture/notifications.md#authorization)
- [Architecture: agyn CLI — Thread Commands](../architecture/agyn-cli.md#thread-commands)
- [Architecture: agyn CLI — Agent and Instance Handles](../architecture/agyn-cli.md#agent-and-instance-handles)
- [Architecture: agyn CLI — Class vs. Instance in Thread Commands](../architecture/agyn-cli.md#class-vs-instance-in-thread-commands)
- [Architecture: agynd CLI — Platform Connection](../architecture/agynd-cli.md#1-platform-connection)
- [Architecture: agynd CLI — Message Formatting](../architecture/agynd-cli.md#2-message-formatting)
- [Architecture: Agent — Communication Protocol](../architecture/agent/overview.md#communication-protocol)
- [Architecture: Agent State](../architecture/agent/state.md)

## Delta

### Agents Service — Instances and Inbox

- `agent_instances` table does not exist. `CreateInstance`, `GetInstance`, `ListInstances`, `PauseInstance`, `ResumeInstance`, `DeleteInstance` do not exist.
- Instance identity registration with the Identity service does not exist (no `identity_type=agent_instance` writer).
- `inbox_items` table does not exist. `WriteInboxItem`, `GetUnackedInboxItems`, `AckInboxItems`, `GetUnackedInboxCount`, and the internal `FanoutInboxItem` RPC do not exist.
- Instance idle GC (background loop that transitions `active → paused` past `instance_idle_ttl`) is not implemented.
- `instance.updated` and inbox-scoped `message.created` events on `instance_inbox:{id}` are not emitted.

### Threads Service

- `Participant.identity_id` is treated as a generic identity id with no class/instance distinction. There is no rewrite of an agent class id to a fresh instance id on `CreateThread` / `AddParticipant`.
- `SendMessage` writes a `MessageRecipient` row for every non-sender participant regardless of identity type. There is no branch that writes an inbox item for `agent_instance` participants.
- `SendMessage` publishes `message.created` only to `thread_participant:{participant_id}`. It does not publish to `instance_inbox:{instance_id}` for agent participants.
- `Participant.passive` field still exists (from the prior [2026-04-12 change](2026-04-12-agent-to-agent-communication.md)). This flag must be removed together with the class-on-add rewrite becoming the routing mechanism for the previously-passive case.
- The Agent Availability Check reads `identity_type == agent` only. It needs to also handle `agent_instance` by resolving the class id.

### Identity Service

- `identity_type` enum does not include `agent_instance`.
- `org_nicknames.instance_suffix` column does not exist.
- `ResolveNickname` does not parse `@nickname#suffix`. Handles containing `#` are rejected or misresolved.
- No caller registers instance identities.

### Notifications Service

- `instance_inbox:{id}` room type is not defined. Publish is allowed by Istio anyway (no room whitelisting), but no producer emits to it and no consumer is expected to subscribe.
- The `:me` sentinel rewrite covers only `thread_participant:`. It does not rewrite `instance_inbox:me`.
- Room authorization has no rule for `instance_inbox:{id}`.

### Agents Orchestrator

- Reconciliation desired-state query reads unacked messages from Threads and reconciles on `(thread_id, agent_id)` pairs. It does not list instances with unacked inbox items from the Agents service.
- Workload records in the Runners service are keyed by `(thread_id, agent_id)`. Migration to keying by `instance_id` (with `thread_id` still recorded on the workload for observability) is not done.
- `Runners.ListWorkloadsByInstance` and `Runners.ListVolumesByInstance` do not exist. Callers of `ListWorkloadsByThread` / `ListVolumesByThread` in the Start Decision and Runner Selection code paths need to switch.
- Start-failure terminal escape calls `Threads.DegradeThread`. It needs to call `Agents.PauseInstance` instead so the failure does not tear down the thread — other participants (users, other agents) may still be productive on it.
- Workload spec assembly injects `THREAD_ID`. It needs to inject `INSTANCE_ID` (workload scope) and `AGENT_ID` (class for config fetch). `THREAD_ID` is removed from the platform-managed env set.
- Idle timeout only stops workloads. There is no instance-level `instance_idle_ttl` enforcement (the Agents Service owns it in the new model — the Orchestrator only owns workload-level idleness).

### agynd

- Subscribes to `thread_participant:me`. Needs to switch to `instance_inbox:me`.
- Reads via `Threads.GetUnackedMessages(thread_id: THREAD_ID)`. Needs to switch to `Agents.GetUnackedInboxItems(instance_id: INSTANCE_ID)`.
- Acknowledges via `Threads.AckMessages`. Needs to switch to `Agents.AckInboxItems`.
- Reads `THREAD_ID` from env. Needs to read `INSTANCE_ID` (for inbox/workload calls) and continue using `AGENT_ID` (for class config).
- Message formatting has no per-item `thread:` / `from:` header — items are fed as bare body + optional file URIs. Needs the tagged header so the LLM can address replies.

### agyn CLI

- `agyn threads send`, `read`, and `add` accept a bare command with no `--thread` when `THREAD_ID` env var is present. `THREAD_ID` fallback needs to be removed; `--thread` is mandatory on `send`.
- `--ref` flag on `create` is used for the local ref alias. The spec now uses `--thread` for the ref parameter on `create` for consistency across commands — either rename `--ref` to `--thread` or keep `--ref` as an alias for backward compat.
- `agyn threads reply --to-message MSG_ID` does not exist.
- `--passive` flag on `create` and `add` still exists. Needs to be removed.
- `agyn agents instantiate @class [--label LABEL]` command does not exist.
- Nickname resolution accepts only `@nickname`. Needs to accept `@nickname#suffix` and resolve to an instance.
- The current-agent auto-participation (agent joining its own thread as its instance) is not implemented — currently the caller must add itself explicitly with the `--passive` mechanism.

### Runners Service

- Workload records store `agent_id` and `thread_id`. They need to also store `instance_id` and be queryable by it (`ListWorkloadsByInstance`, `ListVolumesByInstance`).
- Volume records are similarly thread-keyed; they need to become instance-keyed (with `thread_id` retained for observability but no longer the primary lookup key).

### Agent Implementation (`agn`)

- `agn` receives messages as plain text with no thread-source header. It needs to accept the tagged inbox item format from `agynd` and record `thread_id` alongside every inbound message and outbound reply in its internal state.
- Outbound sends assume a single "current thread" implicit in `agynd`'s scope. Every outbound send in `agn`'s tool surface needs to specify the target `thread_id` explicitly. The tool that maps to `Threads.SendMessage` needs a required `thread_id` parameter.
- State layout is keyed per-workload/per-thread. Needs to move to per-instance so state survives workload restarts and spans multiple threads the instance participates in.

### Authorization (`authz`)

- `agent_instance` OpenFGA type does not exist. Instance authorization currently piggybacks on the class `agent` type where needed; a proper `agent_instance` type (with derivations from the class) needs to be defined.
- No permission for direct inbox writes by apps (e.g., `inbox:write` on org or scoped to a class).

### Chat Service

- `activity_status` derivation reads workload rows keyed by `(thread, agent)`. Needs to read by `(thread, agent_instance)` — the instance is the workload scope now.
- Fresh-chat-creates-fresh-instance pattern (user opens a new chat with `@bob` → thread created with class `@bob` → rewritten to a new instance) needs to be reflected in the chat-create flow so the UI can show the correct instance handle.

## Acceptance Signal

- Creating a thread with `agyn threads create --thread research --add @bob --message "..."` returns the resolved participant handle `@bob#7a2f` (system-generated suffix) and creates a fresh `agent_instance` under the hood.
- Adding an existing instance to a new thread (`agyn threads add --thread other --participant @bob#7a2f`) does not create a new instance; a subsequent `SendMessage` on either thread produces exactly one inbox item on `bob#7a2f`'s inbox.
- An agent instance A running on thread X can `agyn threads create --thread sub --add @c --message "..."` and receive `@c#N`'s reply as an inbox item on the same running workload — the LLM sees the reply tagged with the sub thread's id, without a new workload spawn on the sub thread.
- Two threads that both include the same instance route their `SendMessage` fan-out into the same inbox in global-FIFO acceptance order.
- `agyn threads send` without `--thread` fails with an error even inside an agent workload. Setting `THREAD_ID` in the env has no effect.
- An app with `inbox:write` on the org can call `WriteInboxItem(instance_id, ...)` and the instance's workload wakes to process the item tagged with `source: direct` (no `thread_id`).
- Removing a class does not remove existing instances; existing instances continue to process their inboxes with their frozen configuration. (Or: instance-delete is triggered as part of class-delete cleanup — decide during implementation. Note in follow-up.)
- Killing a workload and starting a new one for the same instance mounts the same state volume; agent state (conversation summaries, tool state) is preserved across restarts.
- `agyn agents instantiate @bob --label planning-run-1` creates an active instance with handle `@bob#planning-run-1` and no thread participation. A subsequent `WriteInboxItem` or `AddParticipant` targeting that handle wakes it.
- The Orchestrator no longer degrades threads on start-failure exhaustion; it pauses the instance. `ResumeInstance` (after a config fix) unblocks it on the next tick.

## Notes

- **Supersedes [2026-04-12 Agent-to-Agent Communication](2026-04-12-agent-to-agent-communication.md).** The passive-participant model introduced there is replaced entirely. The still-open deltas from that change (nickname index, `@nickname` resolution, `agyn threads *` command group) remain — but the `--passive` flag, `Participant.passive` column, and orchestrator's non-passive filter are removed by this change instead of implemented.
- **Multi-thread state key.** State is now per-instance, not per-`(agent, thread)`. Any in-flight implementation of the earlier per-`(agent, thread)` volume keying needs to be redirected before it lands, or migrated after.
- **Class deletion semantics.** Open: should deleting a class cascade to its instances? Practical default: prevent class deletion while active instances exist; require explicit instance termination first. Confirm during implementation.
- **`agn`'s internal state schema.** The per-instance move is straightforward for state on disk, but the per-outbound-send `thread_id` parameter is a real API change in `agn-sdk-go`. Coordinate with the SDK work.
- **Migration.** No existing production data in the current model to preserve for this greenfield change; if pre-production data exists at implementation time, treat it as disposable (recreate threads and agents in the new model).
- **Console / UI.** Not covered here — Console needs its own delta for showing instances (list, detail, inbox activity) alongside classes. Track separately.
