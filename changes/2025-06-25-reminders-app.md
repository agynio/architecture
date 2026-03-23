# Reminders App

## Target

- [Reminders](../architecture/reminders.md)

## Delta

The Reminders app does not exist in the current implementation. The old reminder implementation was a built-in tool in platform-server (`remind_me` tool, `RemindersService`), which is being replaced by the new architecture.

## Acceptance Signal

- Reminders app is deployed as a cluster-scoped app.
- App is registered in the Apps Service with slug `reminders`.
- App connects to the platform via OpenZiti (binds `app-reminders` service, dials Gateway).
- Agents can create, list, cancel, and get reminders via `agyn app reminders ...`.
- Reminder firing delivers a message to the target thread via SendMessage.
- Pending reminders survive app restart (re-fire missed reminders).

## Notes

- Depends on: Apps concept (`2025-06-25-apps-concept.md`), non-participant senders in Threads (`2025-06-25-threads-non-participant-senders.md`).
- The old `remind_me` tool and `RemindersService` in platform-server are deprecated once this is operational.
