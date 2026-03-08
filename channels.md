# Channels

## Overview

Channels are the bidirectional interface connecting 3rd-party products and Agyn's own apps with the Threads service. Each channel translates between an external messaging protocol and the internal thread model.

## Responsibilities

A channel has two sides:

1. **Configuration** (control plane) — Defines the desired state of the channel connection: credentials, target identifiers, routing rules.
2. **Connection** (data plane) — Maintains the live connection to the 3rd-party API, translates messages in both directions.

## Message Flow

```
3rd-party App (e.g., Slack)          Agyn Platform
─────────────────────────────────────────────────────────
                                     
User sends message ──► Channel ──► Thread ──► Agent
                        (inbound)         processes
                                          message
Agent replies    ◄── Channel ◄── Thread ◄── Agent
                      (outbound)          posts response
                                     
Notification ◄─────────────── Notifications
(to external)                 service
```

### Inbound Flow

1. External event arrives (e.g., Slack message via Socket Mode).
2. Channel service translates the event into a thread message.
3. Channel sends the message to the Threads service.
4. Thread now has a pending message, which triggers the Agents orchestrator.

### Outbound Flow

1. Agent posts a response to the thread.
2. Notifications service emits a new-message event.
3. Channel receives the notification (subscribed to relevant thread rooms).
4. Channel translates the message back to the external format and sends it to the 3rd-party API.

## Channel Interface

Every channel implementation (Slack, web app, mobile app, etc.) must implement the same channel interface. Agyn's own web and mobile apps are also channel implementations.

## Channel Configuration

Channel configuration is managed by the control plane and includes:

| Field | Description | Example (Slack) |
|-------|-------------|-----------------|
| Type | Channel type identifier | `slack` |
| Credentials | Authentication for the 3rd-party API | Bot token (`xoxb-...`), App token (`xapp-...`) |
| Target | External resource identifier | Slack channel ID |
| Routing | Rules for mapping external conversations to threads | Thread-per-channel, thread-per-Slack-thread |

The control plane reconciles channel connections: if credentials rotate or configuration changes, the channel service reconnects.
