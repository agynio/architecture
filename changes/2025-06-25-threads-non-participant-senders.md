# Threads Non-Participant Senders

## Target

- [Threads — Non-Participant Senders](../architecture/threads.md#non-participant-senders)

## Delta

The Threads service currently requires `sender_id` to be a thread participant when calling `SendMessage`. App identities need to send messages without being participants.

## Acceptance Signal

- `SendMessage` allows `sender_id` with `identity_type: app` without participant membership.
- Authorization check confirms the app has `thread:write` permission.
- `MessageRecipient` rows are created for all thread participants (sender not excluded since sender is not a participant).

## Notes

- Required by: Reminders app, future write-only apps (GitHub).
- No change to participant-based sending — existing behavior for users, agents, and participant apps is unchanged.
