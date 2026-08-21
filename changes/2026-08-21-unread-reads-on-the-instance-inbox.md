# Thread Read State for Agent-Instance Participants

## Target

- [Threads — MessageRecipient](../architecture/threads.md#messagerecipient)
- [Threads — Message Delivery](../architecture/threads.md#message-delivery)
- [Agent Instances — Routing rules](../architecture/agent-instances.md#routing-rules)
- [agyn-cli — Unread and Acknowledgment](../architecture/agyn-cli.md#unread-and-acknowledgment)

## Delta

`MessageRecipient` rows are written only for `user` and `app` participants. An `agent_instance` participant gets an [inbox item](../architecture/agent-instances.md#inbox) and nothing else, so `GetUnackedMessages` — which reads those rows — returns empty for an instance caller regardless of what arrived on the thread. Unread is unanswerable for exactly the participants that do most of the reading.

The cause is that one mechanism is being asked to carry two concerns. An inbox item is *delivery*: it wakes a runtime, survives it not running, and is redelivered until processed. "What have I not read on this thread" is a *reader's cursor*. Instances were given the first and left without the second.

- `agyn threads read --thread REF --unread` inside an agent workload returns nothing. The [Example Flow](../architecture/agyn-cli.md#example-flow) — "the agent's next invocation reads the messages" — is this case.
- `read --unread --wait` compounds it. The wake half is correct, so the notification arrives, the re-read returns nothing, and the command blocks to timeout and exits 1.

## Acceptance Signal

- `SendMessage` writes a `MessageRecipient` row for every non-sender participant, `agent_instance` included. Delivery is unchanged: instances are woken through their inbox and `instance_inbox:` room, users and apps through `thread_participant:`.
- `GetUnackedMessages`, `GetUnackedMessageCounts` and `AckMessages` read and write those rows with no branch on identity type, and return `Message` as they do today. An instance caller gets the same answers a user would.
- The row written for an instance gates nothing. Redelivery, the Orchestrator's reconciliation set and instance lifecycle continue to key off the inbox item's `acked_at`. `AckMessages` never touches an inbox item; `AckInboxItems` never marks anything read.
- Threads never reads the inbox. `FanoutInboxItem` stays write-only and no read RPC is added to the Agents Service.
- The `agyn` CLI's read paths carry no notion of an inbox and no branch on caller identity type.
- Messages are marked read by whatever hands them to a reader: the CLI after `--unread` returns, and `agynd` via `Threads.AckMessages(message_ids)` for the thread messages it fed into a turn, at the same point it acks the turn's inbox items. `direct` items carry no `message_id` and mark nothing. `AckMessages` is idempotent, so both markers may cover the same message.
- Arrival alone marks nothing. The messages that woke an instance are unread for the duration of its turn, and an interrupted turn leaves both the inbox items unacked and the messages unread.
- The [Example Flow](../architecture/agyn-cli.md#example-flow) runs as written inside an agent workload, and `read --unread --wait` returns the woken message rather than timing out.

## Notes

Two acks exist for an instance and they answer different questions at the same moment. `AckInboxItems` says the runtime processed the item — it releases redelivery and the reconciliation set. `AckMessages` says the agent was shown the message. `agynd` makes both calls at turn end; nothing else couples them, and `direct` inbox items participate in the first only.

This reverses the "no double-write" rule in [Routing rules](../architecture/agent-instances.md#routing-rules). That rule was written to stop two *delivery* paths from racing to feed one instance, and delivery stays single-path. What it also removed, unintentionally, was the instance's read cursor. Restoring the row costs one insert per instance participant per message and answers a question the inbox cannot: the inbox knows what the runtime processed, never what a reader has seen.

Keeping the two separate is what makes both correct. `agynd` acks after a turn, which is the wrong moment to call a message read — an agent may process an item without any reader having looked at the thread, and a shell in a later turn may read messages whose items were acked long ago. Answering `--unread` from ack state would have made the CLI's ack a no-op, made `--unread` non-advancing inside a turn, and put a cross-service read into a thread query. None of those arise here.

An instance that never calls `AckMessages` accumulates unacked rows nobody consults. That is inert, and it is the correct default for an agent that reads with `--tail` or `--after`, or never reads at all.

Notification rooms remain identity-typed (`thread_participant:me` vs `instance_inbox:me`, see [Wait Behavior](../architecture/agyn-cli.md#wait-behavior)). Those are delivery, which is the half that legitimately differs.
