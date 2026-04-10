# Port Exposure

## Purpose

Port Exposure allows users to access development servers and other network services running inside agent containers directly from their own machines. An agent starts a service (e.g., a dev server on port 3000), exposes it through the platform, and shares a link. The user opens the link in their browser and interacts with the service as if it were running locally.

## User Stories

- As a user, I want to enroll my device into the platform network so I can access services exposed by agents.
- As an agent, I want to expose a port and receive an access URL so I can share it with the user.
- As a user, I want the exposed service to be cleaned up automatically when the agent stops so I don't have stale connections.

## Prerequisites

The user must install a Ziti tunnel client on their machine and enroll it using a JWT token generated from the Console. See [Devices](#devices) for the enrollment flow. Once enrolled, the device is part of the OpenZiti network and can access any exposed service.

## Flow

1. The user enrolls their device into the platform network via the Console (one-time setup).
2. The user sends a message to a conversation with an agent.
3. The platform starts the agent.
4. The agent executes work and starts a dev server (e.g., on port 3000).
5. The agent exposes the port via `agyn expose add 3000`.
6. The agent receives the access URL (`http://exposed-<id>.ziti:3000`) and shares it with the user (e.g., posts it to the conversation).
7. The user opens the link in their browser. The Ziti tunnel on the user's machine resolves the hostname and routes traffic to the agent container over the OpenZiti network.

The platform does not automatically post the link — the agent decides when and how to share it.

## Link Format

Exposed services are reachable at:

```
http://exposed-<id>.ziti:<port>
```

Where `<id>` is the unique identifier of the exposure and `<port>` is the exposed port number. The `.ziti` suffix is resolved by the user's Ziti tunnel — it is not a public DNS name.

## Lifecycle

- Ports are exposed on demand by the agent during execution.
- Ports can be explicitly un-exposed by the agent via `agyn expose remove 3000`.
- All exposed ports are automatically cleaned up when the agent workload is stopped (idle timeout, conversation resolved, or manual stop).

## Devices

Users enroll their machines into the platform network through the Console. The Devices section is in the user menu (above API Tokens).

### Enrollment Flow

1. User opens the Console and navigates to Devices in the user menu.
2. User clicks "Add Device" and provides a device name (e.g., "Work Laptop").
3. The platform generates an enrollment JWT token.
4. The user copies the JWT and uses it in their Ziti tunnel client (Ziti Desktop Edge or `ziti-edge-tunnel`) to enroll the device.
5. Once enrolled, the device appears as "Enrolled" in the Devices list.

### Device Management

The Devices section shows all registered devices for the user:

| Column | Description |
|--------|-------------|
| Name | User-provided device name |
| Status | `pending` (JWT generated, not yet enrolled) or `enrolled` |
| Created | When the device was registered |

Actions:
- **Delete** — removes the device and revokes its network identity. Requires confirmation.

The enrollment JWT is shown once at creation time and cannot be retrieved again. If the user loses the JWT before enrolling, they must delete the device and create a new one.

## Constraints

- The user must have a Ziti tunnel client installed and running on their machine.
- Exposed services are accessible to any identity connected to the OpenZiti network (enrolled devices, agents, runners, etc.).
- The link uses HTTP (not HTTPS) — TLS termination is not provided by the platform for exposed ports.

## Related Architecture

- [Expose Service](../../architecture/expose-service.md)
- [Users — Devices](../../architecture/users.md#devices)
- [OpenZiti Integration](../../architecture/openziti.md)
- [Console](../../architecture/console.md)
