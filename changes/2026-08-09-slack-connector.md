# Slack Connector

## Target

- [Slack Connector](../architecture/apps/slack-connector.md) (new)
- [Apps — Examples](../architecture/apps.md#examples)
- [Apps — Permissions](../architecture/apps.md#permissions-1)
- [Services](../architecture/services.md)
- [System Overview](../architecture/system-overview.md)

## Delta

**There is no Slack surface.** The only bidirectional 3rd-party bridge is the [Telegram Connector](../architecture/apps/telegram-connector.md), whose model — one chat, one thread, the bot sees every message — does not carry over. Slack conversations happen in threads nested under a channel, the message that starts a thread is usually one the bot never saw, and a bot is invited into a conversation mid-flight by being mentioned.

This change adds **`agynio/slack-connector`**, a participant app that connects a Slack bot to a configured agent.

### Mention-driven, not mirrored

The connector subscribes to `app_mention` only. It does not subscribe to `message.*`, and Slack messages that do not mention the bot never reach the platform.

That is a design constraint, not a simplification. Every `SendMessage` to a thread creates an [inbox item](../architecture/agent-instances.md#inbox) for each agent-instance participant, and an unacked inbox item keeps the instance in the Orchestrator's reconciliation set — so mirroring a channel would start an agent workload for every human message in it. There is no non-waking inbox item today, and once an agent is a participant there is no way to remove it, so a full mirror is a one-way door into that behavior.

### Context assembly

Because the connector is not mirroring, it reconstructs what the agent needs at mention time by reading Slack's own history with `conversations.replies`, and forwards two platform messages per mention: a context message, then the mention itself.

- **The thread root is re-sent on every mention.** It is what the thread is about, it is one message, and the agent cannot fetch it back once its own context has compacted.
- **Replies are a delta** — everything after `last_forwarded_ts`, excluding the bot's own posts.
- **Omissions are always marked.** Context bounds (50 replies / 20,000 chars / 4,000 chars per block) drop the oldest replies, and the agent is told what was dropped rather than handed a silently partial view.
- **The agent joins before anything is forwarded.** Inbox items are created only for participants at `SendMessage` time and adding a participant does not backfill, so create-thread → add-agent → forward is the required order.

### History access can fail on a mention that succeeded

A bot can be mentioned in a channel it is not a member of, but it cannot read that channel's history. The connector auto-joins public channels on `not_in_channel` and, when that is not possible, forwards the mention with an explicit "thread history unavailable" banner and posts the same notice into Slack.

### Block and attachment rendering

An alert's top-level `text` is typically a fallback like `[FIRING:1] HighErrorRate`. The values, labels and runbook links live in Block Kit blocks or — for Grafana, Alertmanager, PagerDuty and Datadog — in legacy `attachments`. The spec defines rendering for both, plus inline entity resolution (`<@U…>`, `<#C…|…>`, `<url|label>`), mrkdwn-to-markdown conversion, and sender attribution for webhook and bot messages.

### Mapping

`(installation_id, team_id, channel_id, thread_ts) → thread_id`, with top-level mentions normalized to `thread_ts = event.ts` so every mention resolves to a Slack thread. One Slack thread maps to one platform thread maps to one agent instance.

### Connectivity

Socket Mode, not Events API webhooks — apps dial out over OpenZiti and have no public HTTPS ingress. One connection per installation, envelope acked within 3 seconds, redelivery deduplicated on `event_id`.

## Acceptance Signal

- `agynio/slack-connector` exists, enrolls as an app with `thread:create` and `participant:add`, and is installable with `bot_token`, `app_token`, `agent_id` and optional `allowed_channels`.
- One Socket Mode connection runs per installation; the 60-second reconciliation loop opens and closes connections as installations appear and disappear.
- Only `app_mention` events are subscribed. No `message.*` subscription exists.
- Envelopes are acked within 3 seconds and events are deduplicated on `(installation_id, event_id)`.
- Mentions from channels outside a non-empty `allowed_channels` are discarded without creating a mapping.
- A mention with no `thread_ts` creates a Slack thread rooted at that message; a mention inside a thread reuses it. Both resolve to one mapping row.
- On first mention the connector creates the platform thread, adds the configured agent class, and only then forwards — verified by the agent's inbox containing both forwarded messages.
- Each mention forwards exactly two platform messages — context, then mention — and the context message is omitted when the root is the mention itself.
- The rendered root appears on every mention; replies appear only when newer than `last_forwarded_ts`; the bot's own posts never appear in the delta.
- `last_forwarded_ts` advances only after both messages are accepted.
- Context bounds are enforced and every omission carries a visible marker.
- `not_in_channel` in a public channel triggers `conversations.join` and one retry; failure past that forwards the mention with the history-unavailable banner, posts the notice to Slack, and audits `history_unavailable`.
- Block Kit blocks and legacy attachments both render to markdown per the spec tables, including `section.fields`, `actions` button links, `rich_text_preformatted`, and attachment `fields`/`title_link`/`color`.
- `<@U…>`, `<#C…|…>`, `<!subteam^…>`, `<url|label>` and HTML entities are resolved; mrkdwn is converted to markdown in both directions.
- Slack files are downloaded with a bearer token, uploaded to Files, referenced as `agyn://file/<id>` in the transcript, and listed in the message's `files` field.
- Outbound messages post with the mapping's `thread_ts`, split above 3,500 characters, and upload `agyn://file/<id>` inline images and attached files via `files.uploadV2`.
- Reactions track progress: 👀 on forward, ✅ on first delivery, ❌ on forward failure. Reaction API failures do not fail the turn.
- `429` responses honor `Retry-After`; `channel_not_found` and `is_archived` mark the mapping dead; `invalid_auth` reports `Misconfigured` and closes the connection.
- A `thread degraded` error rotates the mapping to a new platform thread, and the next mention re-forwards the full Slack thread from the root.
- `ReportInstallationStatus` reports one of `Healthy`, `Degraded`, `Misconfigured`, `Stopped` with the metrics listed in the spec, on state transitions and a 60-second tick.
- `AppendInstallationAuditLogEntry` is called for each event in the spec table with the listed severity and an `idempotency_key`. Per-message events are not written.

## Notes

Depends on [2026-07-14-agent-instances-and-inboxes](2026-07-14-agent-instances-and-inboxes.md) — the class-on-add rewrite and per-instance inbox are what make one Slack thread map to one agent instance.

Two things are deliberately out of the initial version and are the most likely first follow-ups:

- **An app-proxy command surface** (`agyn app slack get-thread`, channel history, search) would let the agent read past the context bounds and re-fetch the root on demand. Its absence is why the root is re-sent on every mention rather than reduced to a one-line header, and why dropped replies are unrecoverable within a turn.
- **Cross-thread memory.** Every alert gets an agent instance with no knowledge of the previous ones. Volumes are provisioned per instance, so this is not solvable by instance scoping — it needs a durable store shared across instances of the class, and is being handled separately.

Slack user identities are not mapped to platform identities, so the [Agent Availability Check](../architecture/threads.md#agent-availability-check) evaluates against the connector's app identity rather than the person who typed the mention. `allowed_channels` is the only connector-side control over who can reach the agent. A private agent additionally requires an agent `owner` to grant the app a role via `SetAgentRole`.
