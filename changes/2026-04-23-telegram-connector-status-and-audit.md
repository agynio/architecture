# Telegram Connector Status and Audit Log

## Target

- [Telegram Connector — Installation Status and Audit Log](../architecture/apps/telegram-connector.md#installation-status-and-audit-log)

## Delta

The Telegram Connector does not report operational state to the platform. Org admins have no visibility into whether a bot is polling, whether its token is valid, or why messages are not being delivered.

Specifically:

- Connector never calls `ReportInstallationStatus` — installation status is always absent.
- Connector never calls `AppendInstallationAuditLogEntry` — the audit log is always empty.
- No per-installation metrics (active chats, blocked chats, inbound/outbound counts, last update / last outbound timestamps) are collected or exposed.
- Telegram-side failures (`401 Unauthorized` on the bot token, sustained `getUpdates` outages, exhausted outbound retries) are not surfaced anywhere outside the connector's own logs.
- Thread-mapping rotations after `thread degraded` errors are not recorded.

## Acceptance Signal

- Connector sets `ReportInstallationStatus` on startup of each installation's polling loop and refreshes it on a 60-second tick and on state transitions.
- Status headline is one of `Healthy`, `Degraded`, `Misconfigured`, `Stopped`, matching the spec table.
- Status body includes `last_update_at`, `last_update_id`, `active_chats`, `blocked_chats`, `inbound_messages_1h`, `outbound_messages_1h`, `last_outbound_at`, and `last_error` (when applicable), with absolute UTC timestamps.
- Status is cleared (empty string) when a polling loop stops because the installation was removed.
- Connector appends audit entries for each event listed in the spec table (`polling_started`, `polling_stopped`, `configuration_invalid`, `bot_token_rejected`, `telegram_unreachable`, `telegram_recovered`, `thread_degraded_rotated`, `outbound_delivery_failed`, `bot_blocked_by_user`, `bot_unblocked_by_user`) with the specified severity.
- Per-message events are not written to the audit log.
- Audit entries are written with an `idempotency_key` so retries after Gateway errors do not duplicate entries.
- `bot_blocked_by_user` is emitted once per transition into the blocked state, not on every subsequent send attempt.

## Notes

Depends on [2026-04-20-installation-status-and-audit-log](2026-04-20-installation-status-and-audit-log.md) — the platform APIs (`ReportInstallationStatus`, `AppendInstallationAuditLogEntry`) must exist before the connector can call them.
