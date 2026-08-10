# Port Exposure

## Purpose

Port Exposure allows users to access development servers and other network services running inside agent containers and [sandboxes](../sandboxes/sandboxes.md) directly from their own machines. An agent starts a service (e.g., a dev server on port 3000), exposes it through the platform, and shares a link. The user opens the link in their browser and interacts with the service as if it were running locally.

## User Stories

- As a user, I want to enroll my device into the platform network so I can access services exposed by agents.
- As an agent, I want to expose a port and receive an access URL so I can share it with the user.
- As a user, I want a link to tell me which sandbox or which agent is serving it, so I know what I am opening before I open it.
- As an engineer, I want the link to my sandbox's dev server to stay the same across restarts, so I can bookmark it and share it once.
- As a user, I want the exposed service to be cleaned up automatically when the agent stops so I don't have stale connections.

## Prerequisites

The user must install a Ziti tunnel client on their machine and enroll it using a JWT token generated from the Console. See [Devices](#devices) for the enrollment flow. Once enrolled, the device is part of the OpenZiti network and can access any exposed service.

## Flow

1. The user enrolls their device into the platform network via the Console (one-time setup).
2. The user sends a message to a conversation with an agent.
3. The platform starts the agent.
4. The agent executes work and starts a dev server (e.g., on port 3000).
5. The agent exposes the port via `agyn expose add 3000`.
6. The agent receives the access URL (`http://research.bob.acme.agyn:3000`) and shares it with the user (e.g., posts it to the conversation).
7. The user opens the link in their browser. The Ziti tunnel on the user's machine resolves the hostname and routes traffic to the agent container over the OpenZiti network.

The platform does not automatically post the link — the agent decides when and how to share it.

The same command works at a sandbox shell, and the flow is otherwise identical: the engineer runs `agyn expose add 3000` themselves and opens the resulting link.

## Link Format

An exposed service is reachable at an address that names whoever is serving it:

| Served by | Address | Example |
|---|---|---|
| A [sandbox](../sandboxes/sandboxes.md) | `http://<sandbox>.<organization>.agyn:<port>` | `http://super-sandbox.acme.agyn:3000` |
| An agent instance | `http://<instance>.<agent>.<organization>.agyn:<port>` | `http://research.bob.acme.agyn:3000` |

The agent form is the agent's `@mention` handle read back to front: `@bob#research` serves at `research.bob`. An agent instance nobody has labelled carries a short generated tag instead (`7a2f.bob.acme.agyn`) — still enough to see which agent it is.

The organization is always in the address. It is what keeps two teams' `super-sandbox` apart, and what tells you whose service you are about to open when a link arrives from someone in another organization.

The `.agyn` suffix is resolved by the user's Ziti tunnel — it is not a public DNS name.

**A link is a name, not a secret.** It is meant to be readable and shareable, which also means it is guessable by anyone already on the platform network. What protects an exposed service is network membership and whatever authentication the service itself requires — never the obscurity of its address.

**The address follows the entity, so links keep working.** Restarting a sandbox or an agent workload and re-exposing the same port gives back the same URL — a bookmark survives. Two things change an address: renaming the sandbox, the agent, or the organization, and deleting a sandbox and creating a new one with the same name, which hands the old address to the new sandbox. Renaming something with live exposures breaks links already shared for it.

## Lifecycle

- Ports are exposed on demand during execution — by the agent, or by a person at a sandbox shell.
- Ports can be explicitly un-exposed via `agyn expose remove 3000`.
- Exposing a port that is already exposed returns the existing link rather than a second one.
- All exposed ports are automatically cleaned up when the workload is stopped (idle timeout, conversation resolved, or manual stop).

## Devices

Users enroll their machines into the platform network through the Console. The Devices section is in the user menu (above API Tokens).

### Enrollment Flow

1. User opens the Console and navigates to Devices in the user menu.
2. User clicks "Add Device" and provides a device name (e.g., "Work Laptop").
3. The platform generates an enrollment JWT token.
4. The user downloads the JWT as a file or copies it. The file name is derived from the device name with a `.jwt` extension (e.g., `work-laptop.jwt`). The JWT is shown once and cannot be retrieved again.
5. The user uses the JWT file or token in their Ziti tunnel client (Ziti Desktop Edge or `ziti-edge-tunnel`) to enroll the device.
6. Once enrolled, the device appears as "Enrolled" in the Devices list.

### Device Management

The Devices section shows all registered devices for the user:

| Column | Description |
|--------|-------------|
| Name | User-provided device name |
| Status | `pending` (JWT generated, not yet enrolled) or `enrolled` |
| Created | When the device was registered |

Actions:
- **Delete** — removes the device and revokes its network identity. Requires confirmation.

The enrollment JWT is shown once at creation time and cannot be retrieved again. The creation dialog offers two ways to save it: copy to clipboard or download as a `.jwt` file named after the device. If the user loses the JWT before enrolling, they must delete the device and create a new one.

## Constraints

- The user must have a Ziti tunnel client installed and running on their machine.
- Exposed services are accessible to any identity connected to the OpenZiti network (enrolled devices, agents, runners, etc.).
- The link uses HTTP (not HTTPS) — TLS termination is not provided by the platform for exposed ports.
- Sandbox names, agent nicknames, and organization slugs appear in links, so they are visible to everyone a link is shared with — including participants from other organizations in a shared conversation.

## Related Architecture

- [Expose Service](../../architecture/expose-service.md)
- [Users — Devices](../../architecture/users.md#devices)
- [OpenZiti Integration](../../architecture/openziti.md)
- [Console](../../architecture/console.md)
