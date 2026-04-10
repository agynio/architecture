# Exposed Ports (Dev Server Links)

## Purpose

Agents often start local development servers while working (web apps, docs sites, API servers). The platform allows an agent to share a running server with the user as a **clickable link**.

The link is reachable only from machines enrolled in the platform's OpenZiti network.

## Prerequisites

Before opening an exposed port link, the user must:

1. Install **Ziti Desktop Edge** (or another OpenZiti tunnel client).
2. Enroll their machine using an **enrollment token** generated in the Console.

## User Stories

- As a user, when an agent starts a dev server, I want to receive a link I can open in my browser.
- As a user, I want exposed links to work from my local machine without SSH, kubectl, or port-forwarding.

## Link Format

Agents share exposed ports using URLs of the form:

```
http://exposed-<id>.ziti:<port>
```

Examples:

- `http://exposed-ab12cd34.ziti:5173`
- `http://exposed-k9x3p2q1.ziti:3000`

## Behavior

1. The user sends a message to an agent in a conversation.
2. The agent starts and performs work.
3. If the agent starts a development server, it shares a link in the conversation.
4. The user opens the link in a browser.
5. The browser loads the server content over the OpenZiti tunnel.

## Failure Modes

- If the user has not enrolled a Ziti identity on their machine, the link is not reachable.
- If the agent stops (workload ends) or the server process exits, the link stops working.
