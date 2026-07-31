# Agent Default Thread and Final Turn Message

## Target

- [Architecture: Agent Instances — Outbound](../architecture/agent-instances.md#outbound)
- [Architecture: Agent Instances — Default thread](../architecture/agent-instances.md#default-thread)
- [Architecture: Agent Instances — Final turn message](../architecture/agent-instances.md#final-turn-message)
- [Architecture: Agent Instances — Entity](../architecture/agent-instances.md#entity)
- [Architecture: Agent Instances — Creation](../architecture/agent-instances.md#creation)
- [Architecture: Agents Service — Agent Instance API](../architecture/agents-service.md#agent-instance-api)
- [Architecture: Threads — Interface](../architecture/threads.md#interface)
- [Architecture: Resource Definitions — Agent](../architecture/resource-definitions.md#agent)
- [Architecture: agynd CLI — Final Turn Message](../architecture/agynd-cli.md#6-final-turn-message)
- [Architecture: agyn CLI — Omitting `--thread`](../architecture/agyn-cli.md#omitting---thread)
- [Architecture: Agent — Overview](../architecture/agent/overview.md)

## Delta

### Agent Instances

- `default_thread_id` does not exist on the instance entity. Nothing records which thread an instance was created to serve, so an instance has no fallback destination for its own messages.
- No specified destination for an agent CLI's final turn text. The [2026-07-14 change](2026-07-14-agent-instances-and-inboxes.md) removed `THREAD_ID` — which had made the workload's single thread the implicit destination — and replaced it with per-send explicit targeting, without a mechanism for a wrapped CLI to express a target. The turn's final output has nowhere to go.

### Agents Service

- `CreateInstance` has no creation `context` object and accepts no explicit `default_thread_id`, and applies no class policy to either. Threads' class-on-add rewrite supplies no `context.thread_id`; `agyn agents instantiate` has no `--default-thread`.
- `GetInstance` does not return `default_thread_id`.
- `SetInstanceDefaultThread` does not exist.

### Threads Service

- `SendMessage` requires `thread_id` unconditionally. Resolving an omitted `thread_id` from an `agent_instance` caller's `default_thread_id` — and rejecting omission for every other caller type and for instances with a NULL default — is unimplemented.

### Resource Definitions

- `default_thread` and `final_message` do not exist on the Agent resource.

### agynd

- Does not read `final_message` from class configuration.
- Does not post the agent CLI's final turn text anywhere. Posting it to the instance's `default_thread_id` after turn completion and before ack — with no-op and log when the text is empty or the default is NULL — is unimplemented.
- Message formatting for `source_kind = direct` items tells the agent it "must decide where to write" with no mechanism behind it.

### agyn CLI

- `--thread` is mandatory on `threads send` for all callers. Omitting it inside an agent workload with a default thread — by omitting the field on the wire and letting Threads resolve it — is unimplemented.
- `agyn agents instantiate` has no `--default-thread` flag.

### Console

- No surface for `default_thread` or `final_message` in agent create/edit, and no display of an instance's default thread. See Notes.

## Acceptance Signal

- A chat with a single agent whose class sets `final_message=default_thread` delivers the agent's reply into the chat thread with no send tool call in the transcript, and [Chat](../architecture/chat.md#activity-status) transitions `running → finished` around it.
- The same agent on `final_message=discard` (the default) posts nothing on turn end; only its explicit sends appear on the thread.
- `agyn threads send --message "..."` with no `--thread` inside an agent workload lands on the instance's `default_thread_id`. The same command run by a user, or by an instance whose default is NULL, is rejected.
- Unsetting `THREAD_ID`, or setting it to an unrelated thread, changes nothing about where an untargeted send lands — resolution is server-side from the caller identity.
- In a chain A → B → C: C's reply to B does not cause B's final message to land on thread BC. With `final_message=default_thread` on B's class, B's final text lands on AB; B reaching thread BC requires an explicit `--thread`.
- An instance created by class-on-add from a class on `default_thread=origin` (the default) has `default_thread_id` equal to the thread that added it, and adding that instance to a second thread leaves the field unchanged.
- The same add against a class on `default_thread=none` produces an instance with a NULL default; its untargeted sends are rejected, and it posts no final message. `agyn agents instantiate --default-thread` against that class still sets the field.
- An instance woken only by an app's `WriteInboxItem` (`source_kind=direct`) posts its final message to its default thread.
- `SetInstanceDefaultThread` succeeds for a caller with `can_manage` and is rejected otherwise.

## Notes

- **Reverses one acceptance signal from [2026-07-14](2026-07-14-agent-instances-and-inboxes.md).** That change asserted `agyn threads send` without `--thread` must fail "even inside an agent workload; no environment variable substitutes for it." The environment-variable half stands — nothing in the container's environment resolves a thread, and `THREAD_ID` stays removed. What changes is that a **server-side** resolution from the caller's own instance identity is now allowed. The original concern was a stale env var in a long-lived container silently misrouting messages; a field on the instance record, set at creation and moved only by an explicit API call, is not exposed to that.
- **The trigger thread is deliberately not the default.** Deriving the destination from the turn's inbox items breaks as soon as agents delegate: in A → B → C, B is woken by C's reply on thread BC while owing its answer to A on thread AB. Origin semantics compose correctly instead, because delegation always creates sub-threads downward.
- **Why `final_message` lives on the class, not the instance.** Whether an agent's final text is a deliverable message or an internal artifact is a property of how the agent is written and prompted — uniform across its instances. The destination varies per instance; the policy does not.
- **`CreateInstance` is the single enforcement point.** Both creation paths funnel through it, and it already holds the class definition, so the `default_thread` policy is applied there rather than in Threads. The [Agents Orchestrator](../architecture/agents-orchestrator.md) is not involved in instance creation at all — it reconciles existing instances with unacked items and starts workloads.
- **Creation circumstances go in a `context` object, not top-level fields.** `context.thread_id` is a fact Threads reports, not an instruction; the class policy decides what it means. Later creation paths will have their own circumstances to report (a source app installation, an initiating identity), and those become new `context` fields rather than a widening top-level request that every caller has to understand.
- **The policy governs the automatic path only.** `none` means "do not infer a destination from whichever thread happened to add this instance," not "this instance may never have a default." `agyn agents instantiate --default-thread` and `SetInstanceDefaultThread` remain deliberate acts. Making `none` a hard lock would need `SetInstanceDefaultThread` to fail conditionally, which buys little over simply not calling it.
- **An enum rather than a boolean.** `final_message` names the thing being configured and matches `availability`, the Agent's other behavior-controlling field. It also leaves room for a further destination (a trigger thread, a fixed thread) without adding a second flag, and keeps the field name free of the destination that a boolean had to encode.
- **Default `discard` and the Chat path.** The default is `discard` so agents that already send explicitly do not post everything twice. The consequence is that a naively created agent dropped into a chat still delivers nothing — which is the break this change closes. The Console's agent create flow should preselect `default_thread` for chat-facing agents; tracked with the Console delta below rather than by changing the field's default.
- **No double-post suppression.** Under `default_thread`, `agynd` posts unconditionally rather than trying to detect that the agent already sent something. An agent legitimately sends to a sub-thread *and* owes an answer on its origin thread, so "already sent" is not a signal that the final message is redundant.
- **Console.** Agent create/edit needs the `default_thread` and `final_message` selectors, and instance surfaces need to show (and allow changing) the default thread. Belongs with the instance Console delta already tracked in [2026-07-14](2026-07-14-agent-instances-and-inboxes.md).
- **`agn-sdk-go` coordination.** Unaffected — the required `thread_id` on `agn`'s outbound send tool stays required. This change concerns the untargeted final text, not tool-driven sends.
