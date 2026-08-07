# Terminal Proxy

## Overview

The Terminal Proxy bridges interactive terminal sessions between clients (the Console's browser terminal, the `agyn` CLI) and the container they target. It is a transparent byte pipe: the client speaks WebSocket, the hosting runner speaks the [`Runner.Exec`](runner.md#execution) bidirectional gRPC stream, and the proxy translates between them without interpreting the bytes.

Primary consumers:

- **[Sandboxes](../product/sandboxes/sandboxes.md)** — `agyn sandbox start`/`connect` and the [Sandboxes App](sandboxes-app.md)'s browser terminal attach shells through the proxy.
- **Console workload inspection** — the terminal on the Workload Detail and Sandbox Detail pages ([Container](../product/concepts.md) terminal access).
- **[Sandbox Workspace Sync](sandbox-sync.md)** — file sync sessions use the same transport in its non-TTY form.

Despite the name, the proxy is not shell-specific: it carries any bidirectional stream to a process in a container. What varies between consumers is the [session kind](#session-kinds), not the transport.

## Classification

| Aspect | Detail |
|--------|--------|
| **Plane** | Data |
| **Language** | Go |
| **Repository** | `agynio/terminal-proxy` |
| **API** | WebSocket (external, via ingress) + gRPC (internal ticket issuance) |
| **State** | None — sessions are in-memory; tickets are self-contained signed tokens validated by any replica |
| **External dependencies** | [Runners](runners.md) (workload → runner resolution, activity reporting), [OpenZiti](openziti.md) (dial runners), [Authorization](authz.md) (checks at ticket issuance), [Agents](agents-service.md) (sandbox session bookkeeping) |

## Session Establishment

Terminal sessions are established in two steps — an authorized ticket issuance over the normal API path, then a WebSocket connect that never carries the caller's long-lived credentials:

```mermaid
sequenceDiagram
    participant C as Client (CLI / Console)
    participant G as Gateway
    participant TP as Terminal Proxy
    participant RS as Runners Service
    participant R as Runner
    participant K as Container PTY

    C->>G: CreateTerminalSession(workload_id, container_name, kind)
    G->>TP: IssueTicket (internal, caller identity from context)
    TP->>TP: OpenFGA check (see Authorization)
    TP->>TP: Resolve kind to a command; validate the kind's parameters
    TP-->>C: { ticket, url }
    C->>TP: WebSocket connect (ticket) + handshake {cols, rows}
    TP->>RS: GetWorkload → runner_id
    TP->>R: OpenZiti Dial runner-{runnerId} → Exec (tty, resize)
    R->>K: allocate PTY, spawn shell
    C-->>K: raw bytes, both directions, until exit
```

1. **`CreateTerminalSession`** (Gateway RPC): the caller requests a session for `(workload_id, container_name, kind)`. The Gateway checks the request is well-formed, resolves the caller's identity from authenticated context, and forwards to the Terminal Proxy's internal `IssueTicket`. Everything that follows from the kind — the authorization check, resolving the kind to a command, validating that kind's parameters — happens in the proxy, which owns terminal sessions; the Gateway routes and does not interpret. `IssueTicket` returns a **ticket**: single-use, bound to the caller's identity and the target, expiring in 30 seconds. Long-lived auth tokens never appear in WebSocket URLs (same reasoning as the [Media Proxy](media-proxy.md)'s avoidance of tokens in `GET` URLs).
2. **WebSocket connect**: the client opens the WebSocket with the ticket, sends a JSON handshake carrying **only the initial terminal size**, and the proxy resolves the hosting runner via `Runners.GetWorkload`, dials it over OpenZiti (`runner-{runnerId}` — see [Runners — Terminal Proxy Integration](runners.md#terminal-proxy-integration)), and opens `Runner.Exec` with TTY enabled.

The command is fixed at ticket issuance, never at attach time. It is derived from the requested [session kind](#session-kinds) and bound into the ticket by the proxy. The WebSocket handshake carries no command — a client cannot escalate beyond what the ticket was issued for.

## Session Kinds

`CreateTerminalSession` takes a **session kind** rather than a command. The proxy holds the command for each kind, so a client never supplies one and the ticket always describes something the platform defined. The mapping lives here rather than in the [Gateway](gateway.md) because it is terminal-session domain logic, and the Gateway routes external requests to internal services rather than interpreting them.

| Kind | TTY | Command | Consumer |
|---|---|---|---|
| `SHELL` | yes | `/bin/sh -lc 'PATH=/agyn/bin:$PATH exec ${SHELL:-sh}'` — see [PATH](#path-in-a-session) | `agyn sandbox start`/`connect`, Console terminal |
| `EXEC` | yes | A caller-supplied command, run with the same `PATH` prefix | Console non-shell inspection commands |
| `SYNC` | no | `/agyn/bin/agyn sandbox sync serve --root <path>` | [Workspace sync](sandbox-sync.md), `agyn sandbox cp` |

`SESSION_KIND_UNSPECIFIED` is rejected with `InvalidArgument`. It does not default to `SHELL` — silently defaulting the field that selects a command is how an unintended command ends up in a ticket.

Authorization does not vary by kind: `EXEC` carries a caller-supplied command but is authorized by the same check as `SHELL` (see [Authorization](#authorization)). What binding the command at issuance buys is not an extra permission gate but immutability — the command cannot be swapped at attach time.

### PATH in a session

The platform's binaries are delivered to `/agyn/bin` by the [binary init containers](agent-init.md), but nothing puts them on `PATH` for a session: `PATH` is extended only by `agynd` when it spawns the agent subprocess, and a sandbox does not run `agynd`. Sessions establish it themselves.

**`SYNC` sidesteps the problem** by using an absolute path, which is what a machine-invoked command should do regardless.

**TTY kinds must prepend after the login profile has run, not before.** The obvious form — setting `PATH` and then exec'ing a login shell — does not work: `/etc/profile` on Debian and Ubuntu assigns `PATH` unconditionally, so a login shell discards anything set ahead of it. The prefix would vanish on exactly the base images most likely to be in use.

The bound command therefore runs the profile first and prepends afterwards:

```sh
/bin/sh -lc 'PATH=/agyn/bin:$PATH exec ${SHELL:-sh}'
```

The outer `sh -l` processes `/etc/profile` and the user's profile; `$PATH` is then whatever that left, and the platform entry goes in front of it. Setting `PATH` through the pod spec is not an alternative — Kubernetes `env` replaces the image's value rather than extending it, and the orchestrator cannot know what the image configured.

The consequence to be aware of: the final shell is **not** a login shell, so shell-specific login files (`~/.bash_profile`) do not run, while `/etc/profile` and `~/.profile` already have. Making the final shell a login shell would re-run the profile and discard the prefix again.

### Root validation

`SYNC`'s `root` parameter is checked in two places, because neither alone can do the job:

| Check | Where | Why there |
|---|---|---|
| Absolute, lexically normalized, no `..` traversal | The proxy, at issuance | Purely textual, and it is what keeps the ticket honest |
| Resolves inside the workload's workspace mount, after following symlinks | The endpoint, at handshake | Only the process inside the container can see the filesystem. Nothing outside it has mount data for a container — `Runners.Container` carries name, role, image, and status — so this cannot be checked before the session exists |

A root failing the second check fails the session at handshake with the resolved path named. This is not a privilege boundary — a sandbox owner can already obtain a shell — but it keeps tickets meaningful and prevents the surface from generalizing into arbitrary remote execution.

`SYNC` sessions run with `Runner.Exec` wall and idle timeouts disabled, as terminal sessions do. A session parked in a blocking poll legitimately emits nothing for the poll interval; an idle timeout would reap healthy sessions. Liveness is the WebSocket ping/pong, unchanged.

Tickets are **self-contained signed tokens** (key shared across proxy replicas), so any replica can validate a ticket issued via any other. Single-use is enforced per replica (best-effort under horizontal scaling); the 30-second expiry and the binding to `(identity, workload, container, command)` bound the replay window.

## Wire Protocol

One WebSocket per session. Frames:

| Frame | Direction | TTY kinds | Non-TTY kinds (`SYNC`) |
|---|---|---|---|
| Binary | client → proxy | Raw input bytes for the PTY (keystrokes, pastes, escape sequences) | Raw stdin bytes — no prefix |
| Binary | proxy → client | Raw PTY output bytes — **no prefix** | **One-byte stream prefix** (`0x00` stdout, `0x01` stderr) followed by the payload |
| Text (JSON) | client → proxy | Control: `{"type":"resize","cols":N,"rows":N}` | Not used |
| Text (JSON) | proxy → client | Control: `{"type":"exit","code":N,"reason":"completed\|cancelled\|error"}` — final message before close | Same |
| Ping/Pong | both | Liveness, every 30s; a peer missing two intervals is considered gone | Same |

The proxy never interprets the *payload*, so one wire protocol serves every session kind — TTY kinds carry PTY bytes, `SYNC` carries the endpoint protocol. The one thing that differs is stream framing on the proxy→client direction.

**Why non-TTY output is tagged.** [`Runner.Exec`](runner.md#execution) distinguishes stdout from stderr (`ExecOptions.separate_stderr`, and distinct response events), but for TTY sessions the proxy writes both as untagged binary frames — correct there, since a PTY merges the two at the kernel anyway. A non-TTY consumer cannot afford that: one byte of stderr from a library inside the container would land inside the protocol stream and corrupt it. Non-TTY kinds therefore request `separate_stderr`, and the proxy prefixes each binary frame with the stream it came from.

## Terminal Semantics

For TTY session kinds, the proxy provides SSH-parity terminal behavior. Explicit guarantees:

- **8-bit clean, zero interpretation.** The data path carries raw bytes. No line buffering, no encoding validation or transformation, no filtering of escape sequences. Colors (16/256/truecolor), alternate screen, cursor addressing, mouse reporting, bracketed paste — anything the container-side program emits and the client-side terminal understands passes through untouched. Full-screen TUIs (`vim`, `htop`, `tmux`) work.
- **A real PTY.** The runner allocates a PTY in the container (Kubernetes `pods/exec` with `tty: true`); the shell runs as a proper foreground process group. Job control, signals-as-bytes (`Ctrl-C` → `^C` → SIGINT via the line discipline), `isatty()` — all behave as on a local terminal.
- **Resize propagation.** The handshake carries the initial size; `resize` control messages are forwarded to the PTY (the k8s exec resize channel), delivering SIGWINCH to the foreground process. The CLI sends one on every SIGWINCH; the Console on every pane resize.
- **No session caps.** Terminal sessions have no wall timeout, no idle timeout, and no output size limit at the exec level (`Runner.Exec` timeouts are disabled for terminal sessions). Session lifetime is bounded only by workload lifetime and explicit policy (a sandbox's own idle timeout counts *detached* time — an attached session keeps it alive indefinitely).
- **Backpressure, not truncation.** Flow control is end-to-end: WebSocket and gRPC stream backpressure propagate to the PTY, which blocks the writer — output is never dropped or truncated by the proxy.
- **Concurrent sessions.** Multiple sessions may target the same container (subject to the same authorization); each gets its own PTY, like multiple SSH connections to one host.
- **Exit propagation.** When the shell exits, the proxy sends the `exit` control message and closes. The `agyn` CLI exits with the same code.

What is deliberately *not* provided: session persistence across disconnects. A dropped WebSocket ends the exec — the PTY closes and the foreground process group receives SIGHUP, exactly like a dropped SSH connection. The container (and anything backgrounded with `nohup`/`setsid`, or running under `tmux`/`screen` from the image) keeps running; `agyn sandbox connect` opens a fresh shell. Server-side session multiplexing is a client-image concern (`tmux`), not a platform feature.

## CLI Behavior (`agyn sandbox start` / `connect`)

- Puts the local terminal into raw mode for the duration of the session and restores termios state on exit, including abnormal exits.
- Forwards SIGWINCH as `resize` control messages; sends the initial size in the handshake.
- Forwards all input bytes verbatim — there is no client-side escape character in v1 (nothing is stolen from the byte stream; detach by ending the shell or closing the terminal).
- Propagates the remote exit code as its own exit code; prints a one-line notice on abnormal session termination (workload stopped, connection lost) to distinguish it from a normal shell exit.

The Console uses xterm.js over the same wire protocol.

## Sandbox Activity Reporting

The Terminal Proxy is the component that knows whether a session is attached, so it owns sandbox idle signaling. The accounting is deliberately **per-session, not per-proxy** — no replica coordination, shared session counts, or sticky routing is needed:

- Each attached **TTY** session to a sandbox workload independently drives `Runners.TouchWorkload` every 10 seconds (the same activity path [`agynd`](agynd-cli.md) uses for agent workloads; concurrent touches are idempotent). The [Agents Orchestrator](agents-orchestrator.md)'s existing idle-timeout machinery then applies unchanged: a sandbox is stopped when `now − last_activity_at` exceeds its `idle_timeout`, which can only happen once **no session on any replica** is touching anymore.
- **`SYNC` sessions never touch the workload.** A background sync session would otherwise keep a sandbox running for as long as a laptop stayed connected, defeating the idle timeout and the metering it bounds. A sandbox therefore idles out from under an active sync session, which pauses until the owner connects again — see [Sandbox Workspace Sync — Reconnection](sandbox-sync.md#reconnection).
- On every TTY session detach, the proxy sets `last_session_at` on the [Sandbox](resource-definitions.md#sandbox) via an internal Agents service RPC (display/bookkeeping only — idle enforcement uses workload activity, so there is no need to detect the "final" detach). `SYNC` sessions do not update it, for the same reason they do not touch the workload.
- A crashed proxy replica needs no cleanup protocol: its sessions' touches simply stop, and the idle clock starts running from the last touch.

Agent workloads get no activity touches from terminal sessions — inspecting an agent container does not keep it alive, and the orchestrator may stop it mid-session (the session ends with `reason: cancelled`).

## Authorization

Checks run in the Terminal Proxy at ticket issuance, against OpenFGA. The Gateway supplies the caller's identity from authenticated context and nothing else — it does not decide who may attach:

| Target | Check |
|---|---|
| Sandbox workload | `can_connect` on the sandbox — its `owner`, or an identity the owner has shared it with — for every session kind including `SYNC`. Org owners can manage sandboxes but **cannot** attach on that basis; they need a share like anyone else. See [Sandboxes — Permissions](../product/sandboxes/sandboxes.md#permissions) |
| Agent workload container | `can_edit_config` on the agent class — a terminal grants nothing a config editor couldn't already obtain by editing the agent's configuration |

The WebSocket itself is authenticated solely by the ticket. Ticket properties: single-use, 30-second expiry, bound to `(identity, workload_id, container_name, kind, command)`.

## OpenZiti Identity

The Terminal Proxy dials runners over the overlay, so it holds an identity of its own. It is an infrastructure service and follows [Service Identity Self-Enrollment](openziti.md#service-identity-self-enrollment): each pod requests an ephemeral identity from Ziti Management at startup, renews its lease on a timer, and terminates if the identity is lost.

The identity is per-pod and is never mounted from a Secret. This service is designed to run several replicas — the ticket signing key is shared precisely so any replica can validate a ticket issued by another — and a mounted identity would be shared by all of them, outliving the pods that used it.

Its identity carries the `terminal-proxy-hosts` role attribute, which the static `terminal-proxy-dial-runners` policy grants against `#runner-services`. See [Runners — Terminal Proxy Integration](runners.md#terminal-proxy-integration).

## Failure Modes

| Failure | Behavior |
|---|---|
| Proxy restart | All sessions drop (stateless service). Clients reconnect via a fresh `CreateTerminalSession` |
| Runner unreachable (Ziti dial fails) | Session fails to establish; client receives a close with an error reason |
| Proxy's Ziti identity lost (lease GC'd) | Pod terminates; the replacement enrolls a fresh identity. See [Identity Loss](openziti.md#identity-loss) |
| Workload stopped mid-session | Exec stream ends; client receives `exit` with `reason: cancelled` |
| Client vanishes (missed pongs) | Proxy cancels the exec (`CancelExecution`), releasing the PTY |

## Related

- [Runner — Execution](runner.md#execution)
- [Runners — Terminal Proxy Integration](runners.md#terminal-proxy-integration)
- [Sandboxes](../product/sandboxes/sandboxes.md)
- [Sandbox Workspace Sync](sandbox-sync.md)
- [OpenZiti Integration](openziti.md)
- [agyn-cli — Sandbox Commands](agyn-cli.md#sandbox-commands)
