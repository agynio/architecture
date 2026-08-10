# One Message per Slack Mention

## Target

- [Slack Connector — Forwarded Message](../architecture/apps/slack-connector.md#forwarded-message)
- [Slack Connector — Mention Handling](../architecture/apps/slack-connector.md#mention-handling)
- [Slack Connector — Required Slack Scopes](../architecture/apps/slack-connector.md#required-slack-scopes)
- [Slack Connector — Agent Authorization](../architecture/apps/slack-connector.md#agent-authorization)

## Delta

**One mention produces two platform messages.** The connector sends the context and then the triggering mention as separate `SendMessage` calls, milliseconds apart.

Both land in the same inbox batch and the agent reads them in one turn, so the split was invisible to the agent — but it is not invisible anywhere a human reads the thread, where the connector appears to answer a mention with two messages. Their order is also decided by comparing creation timestamps that can tie: observed in practice at `08:05:18.376254` and `08:05:18.390731`, rendered by the Chat UI in the wrong order.

The split was meant to keep the instruction unambiguous and last. Being the final section of one message does that just as well.

This change forwards **one** message per mention: the header, the thread root, the reply delta, then a `### Mention` section carrying the triggering message. Sections that have nothing to say are omitted; the header and the `Mention` section are always present.

### Two corrections found alongside it

**`channels:read` and `groups:read` are missing from the required scopes.** `conversations.info` needs them, so channel-name resolution fails and the header renders the raw channel id (`## Slack — C0BNZ29RRRT`) instead of `#name`. Nothing else depends on it — the spec now says the degradation is cosmetic, and names both scopes.

**The Agent Authorization section predates [app membership](../architecture/apps.md#organization-membership).** It said a `private` agent needs a role granted via `SetAgentRole` without saying that an `internal` agent needs nothing, which is now true and is the common case.

## Acceptance Signal

- One `app_mention` produces exactly one `SendMessage`, whatever the thread contains.
- The message opens with the `## Slack — #channel` header and ends with the `### Mention` section; the root and reply sections sit between them.
- The `Thread root` section is omitted when the root is the triggering mention; the header and `Mention` section still appear.
- A rung-3 forward (history unreadable) is one message: header, then the `Mention` section opening with the unavailable banner.
- Every file referenced anywhere in the message — root, replies, or mention — is listed in that one message's `files`.
- `last_forwarded_ts` advances only after the single send is accepted.
- The connector's required scopes include `channels:read` and `groups:read`, and a mention in a channel it can name renders `## Slack — #name`.

## Notes

Found while running the connector against a real workspace. The two messages were visible in the Chat UI in the wrong order, which is what prompted the question — the ordering itself is a Chat UI defect (it ties on a second-granularity timestamp where the stored order is correct to the microsecond), and is worth fixing separately. Sending one message removes the tie rather than relying on that fix.
