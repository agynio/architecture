# Notifications

## Overview

The Notifications service handles real-time event delivery. Its main purpose is to hold persistent connections (sockets) and fan out events to relevant clients.

**Repository:** `agynio/notifications` (Go)

## Architecture

```
Producers                  Notifications Service              Consumers
(platform-server,     ┌──────────────────────────────┐    (web app, mobile,
 agents, channels)    │                              │     channels, etc.)
                      │  ┌────────┐    ┌──────────┐ │
  Publish ──gRPC──►   │  │ Server │───►│  Redis   │ │
                      │  └────────┘    │  Pub/Sub │ │
                      │                └────┬─────┘ │
                      │                     │       │
                      │                ┌────▼─────┐ │    ◄── gRPC Subscribe
                      │                │   Hub    │ │        (internal)
                      │                │ (fan-out)│ │
                      │                └────┬─────┘ │    ◄── Socket.IO
                      │                     │       │        (external)
                      └─────────────────────┘       │
                                                    │
```

## Transport

| Interface | Protocol | Direction |
|-----------|----------|-----------|
| Internal (service-to-service) | gRPC | Publish (unary) + Subscribe (server-streaming) |
| External (client-facing) | Socket.IO | Bidirectional persistent connection |

## gRPC API

Defined in `agynio/api` at `proto/agynio/api/notifications/v1/notifications.proto`.

### Publish

Producers send events to rooms. An event targets one or more rooms and carries a JSON payload.

| Field | Type | Description |
|-------|------|-------------|
| `event` | string | Stable event name (e.g., `message.created`, `job.status`) |
| `rooms` | repeated string | Target rooms (at least one required) |
| `payload` | google.protobuf.Struct | Event-specific JSON payload |
| `source` | string | Origin service identifier |

The server generates an `id` (UUID v4) and `ts` (acceptance timestamp) for each envelope.

### Subscribe

Consumers open a server-streaming RPC to receive all envelopes. Filtering by room is handled client-side or by a gateway layer.

## Envelope

The canonical notification structure:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Server-generated UUID v4 |
| `ts` | timestamp | Server-generated acceptance time |
| `source` | string | Origin service |
| `event` | string | Stable event name |
| `rooms` | repeated string | Target rooms |
| `payload` | Struct | JSON payload (event-specific schema) |

## Internal Design

- **Redis Pub/Sub** is the backbone for distributing envelopes across service instances.
- **Hub** fans out envelopes to all registered subscribers in-process with bounded buffers. Slow consumers are dropped (channel closed) to prevent backpressure.
- Buffer size is configurable per hub instance.
