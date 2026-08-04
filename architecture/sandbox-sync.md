# Sandbox Workspace Sync

## Overview

Workspace sync keeps a directory on an engineer's machine and a directory inside a running [sandbox](../product/sandboxes/sandboxes.md) continuously reconciled in both directions. The engineer edits in their own editor with their own tooling; the agent or shell inside the sandbox sees the same tree, and anything written there appears back on the machine.

Both ends are the **same `agyn` binary**: the CLI runs a sync controller and a local daemon, and the sandbox runs `agyn sandbox sync serve` — available there because the platform's binary init containers put the `agyn` CLI into every workload. There is no second artifact to ship, no agent binary pushed over the wire, and no new network path: sessions ride the [Terminal Proxy](terminal-proxy.md) exec transport that already carries shell sessions.

## Why Not a Network Filesystem

A mounted distributed filesystem is the obvious model and the wrong one for this leg of the problem:

- **Latency dominates.** Every `stat`, `open`, and `readdir` becomes a wide-area round trip. Editor indexing, `git status`, dependency installs, and file watchers issue these by the thousand — the operations that define the experience are exactly the ones a network mount makes unusable.
- **Client mounts are not shippable.** FUSE on macOS requires a kernel extension and, on Apple Silicon, a security-policy change and a reboot. NFS mounts require root. Neither belongs in the path of trying a sandbox.
- **Disconnection is normal.** Laptops sleep and networks drop — the same assumption sandboxes already make. A mount blocks; a sync session keeps working locally and reconciles on reconnect.

Sync gives both ends a real local filesystem at native speed, which is the property the workflow actually depends on.

Shared storage remains the right answer *inside* the cluster — a read-write-many volume mounted by several workloads at once, or the operator's own storage reached through [Private Networks](private-networks.md). Sync is the adapter for the engineer's machine, and composes with either.

## Components

| Where | Component | Responsibility |
|---|---|---|
| Engineer's machine | `agyn sandbox sync` | Creates and inspects sessions; talks to the daemon over a unix socket |
| Engineer's machine | Sync daemon | Owns every session: watching, reconciliation, transfer, reconnection, halt state |
| Engineer's machine | Local endpoint | In-process filesystem access to the local root |
| Gateway | `CreateTerminalSession(kind: SYNC)` | Authorizes and binds the in-sandbox command into a ticket |
| Terminal Proxy | Sync session | A non-TTY session kind that does not report sandbox activity. The wire protocol is unchanged — the proxy does not interpret what it carries |
| Sandbox container | `agyn sandbox sync serve` | Stateless remote endpoint — scans, stages, applies |

## Control Model

The **controller runs on the engineer's machine**. It holds session state, computes reconciliation, and decides every action. The container-side endpoint executes filesystem operations it is told to perform and computes nothing.

This split follows from the sandbox lifecycle rather than from preference. The endpoint process is terminated without warning whenever the WebSocket drops (the proxy cancels the exec), the sandbox idles out, or the TTL fires. Anything authoritative held there is state that is routinely lost. The endpoint therefore persists exactly two things, both disposable: a root marker file and a staging directory.

The **ancestor snapshot** — the merge base that distinguishes "created locally" from "deleted remotely," which are otherwise the same observation — lives only on the engineer's machine.

## Session Establishment

```mermaid
sequenceDiagram
    participant D as Sync daemon
    participant G as Gateway
    participant TP as Terminal Proxy
    participant R as Runner
    participant E as agyn sandbox sync serve

    D->>G: CreateTerminalSession(sandbox, kind: SYNC, root)
    G->>G: OpenFGA check — caller is the sandbox owner
    G->>G: Validate and normalize root; bind command into ticket
    G->>TP: IssueTicket (internal)
    TP-->>D: { ticket, url }
    D->>TP: WebSocket connect (ticket)
    TP->>R: OpenZiti Dial runner-{runnerId} → Exec (tty: false)
    R->>E: spawn endpoint
    D->>E: Handshake / Scan / Poll / Stage / Transition
```

Sync sessions reuse the [Terminal Proxy session establishment](terminal-proxy.md#session-establishment) unchanged, including its authorization: the caller must be the sandbox **owner**. Organization owners can manage a sandbox but cannot attach to one they do not own, and that applies to sync exactly as it applies to shells.

The client requests a **session kind**, not a command. The Gateway holds the command for each kind and binds it into the ticket, so the client supplies no arguments beyond the root path — which the Gateway validates as absolute and normalizes before binding. This keeps the ticket meaningful and stops the surface drifting into arbitrary remote execution.

## Endpoint Protocol

One exec stream per session, with the three standard streams put to distinct use:

| Stream | Carries |
|---|---|
| stdin | Requests from the controller |
| stdout | Responses — protocol frames only |
| stderr | Human-readable diagnostics, surfaced by the CLI on failure |

Frames are length-prefixed protobuf. The exchange is strictly request/response, controller-initiated, one request in flight — there are no server-initiated messages and no multiplexing.

| Request | Purpose |
|---|---|
| `Handshake` | Protocol version, session id, root state and marker, watch mode |
| `Scan` | Walk the root and return a snapshot, given the controller's digest cache |
| `Poll` | Block until the root changes or the timeout elapses |
| `Stage` | Report which content digests are already present, then receive the rest |
| `Transition` | Apply the reconciled changes; return per-path results |
| `Supply` | Stream content the controller needs for the local side |

Change notification is a short-timeout blocking `Poll` rather than an asynchronous push. With a single pipe and no multiplexing there is no way to cancel an in-flight request, so a bounded poll is what returns control to the controller regularly.

`agyn sandbox cp` drives the same endpoint in one-shot form — handshake, one scan, one transfer, one transition, exit. It needs no ancestor, no watching, and no conflict handling, because a copy establishes no ongoing relationship between the two sides.

The endpoint must guarantee that nothing but protocol frames reaches stdout. Any library writing to stdout corrupts the stream, so the real stdout is captured at startup and the process-wide handle redirected to stderr.

## Sync Cycle

1. **Handshake.** The endpoint opens the root, reads or creates its marker, garbage-collects staging directories left by earlier terminated sessions, and reports its watch mode.
2. **Identity check.** The controller compares the marker against its session state. A missing marker where content is expected, or a different one, **halts before anything is read or written**.
3. **Scan.** Both endpoints produce snapshots. The controller supplies its digest cache so only entries whose size or mtime changed are re-hashed. Per-path problems — unreadable files, broken symlinks — are reported, never fatal.
4. **Reconcile.** Three-way reconciliation over ancestor, local, and remote snapshots yields a change list per side plus any conflicts.
5. **Stage.** Content the sandbox needs is written to a staging directory on the same filesystem as the root, after the endpoint reports which digests it already holds.
6. **Transition.** Changes are applied by atomic rename out of staging. A failure on one path does not abort the batch.
7. **Supply.** The reverse direction is fetched, staged, and applied locally.
8. **Commit.** The controller records the new agreed state as the ancestor and returns to `Poll`.

The ancestor is committed only after a complete successful cycle, so an interruption at any point simply recomputes on the next pass. Combined with out-of-place staging and atomic rename, every operation is idempotent and safe to repeat.

## Safety Model

Sync moves and deletes files on a machine the platform does not own. The following are invariants, not defaults:

| Invariant | Rationale |
|---|---|
| A wiped or unrecognized remote root **halts** the session before any read or write | A sandbox terminated by TTL returns an empty `/workspace`. Mirroring that would delete the engineer's work |
| Local deletions are staged into a retained trash directory, not unlinked | A recoverable action needs no confirmation; an unrecoverable one needs a human who may not be present |
| Ownership is never propagated; only the executable bit is | Local uid/gid is meaningless inside the container, and mismatched ownership breaks the sandbox's own tooling |
| Per-path conflicts quarantine the path and leave the session running | Both versions survive, and unrelated work is not held hostage |
| Only session-wide destructive events halt the whole session | Halting must stay rare enough that it carries signal |
| Partially written content never appears at its destination | The endpoint can be terminated at any instruction |

## Halts and Reporting

The daemon **never blocks waiting for a human**. A halt is a durable state in which nothing has been applied, safe to remain in indefinitely — which is what makes it acceptable for a background process that may never be attended to.

Halts are reported through channels ordered by likelihood of being noticed:

| Channel | When it reaches the engineer |
|---|---|
| Inline output | A foreground session is attached |
| Sentinel file in the sync root | Immediately — it appears in the editor tree and in `git status` |
| Native OS notification | At the moment the halt occurs |
| `agyn sandbox sync status`, non-zero exit | On demand, and from shell prompts and editor tasks |
| Banner on `agyn sandbox connect` | The engineer is about to work on those files |

Resolution is always an explicit out-of-band command — see [agyn-cli — Sandbox Sync Commands](agyn-cli.md#sandbox-sync-commands).

## The Local Daemon

Sync must outlive the command that starts it. A session tied to a terminal stops silently when the window closes, which is the failure this design most needs to avoid.

| Concern | Behavior |
|---|---|
| **Start** | Lazily by `agyn sandbox sync`, or explicitly by `agyn sandbox sync daemon start`. No other command starts it — unrelated CLI invocations never spawn a sync daemon |
| **Detachment** | Started in a new session with no controlling terminal, with stdin from `/dev/null` and output to a rotating log. It is reparented to the init process and is unaffected by the terminal, the shell, or the invoking process exiting |
| **Address** | gRPC over a unix socket under `~/.agyn/sync/`, owner-only. A lock file prevents concurrent daemons when two invocations race |
| **Stop** | Explicitly, or automatically once the last session is removed — no idle process outlives its purpose |
| **Reboot** | Nothing resumes automatically. Sessions persist on disk and are listed as not running until sync is started again. Resume-at-login is available as an explicit opt-in that installs a user-level service |
| **Upgrade** | The CLI detects a daemon running an incompatible version on the socket handshake and restarts it. Sessions persist across the restart |

Each session is independent: its own controller loop and its own exec stream. One session halting or disconnecting has no effect on the others.

Session state lives under `~/.agyn/sync/sessions/<id>/` — configuration, the ancestor snapshot, status, and the trash directory.

## Reconnection

The exec stream is expected to break routinely: machine sleep, network loss, sandbox idle stop, TTL termination, and Terminal Proxy restarts, which drop every session by design. The session loop connects, handshakes, syncs, classifies any failure, and reconnects.

| Condition | Response |
|---|---|
| Transport failure (network, proxy restart) | Reconnect with exponential backoff and jitter |
| Sandbox `stopped` | Pause. **The sandbox is not restarted** — a background file change must never start billable compute. Resumes when the engineer next connects |
| Sandbox `terminated` | Halt; the session requires explicit action |
| Root marker mismatch | Halt; nothing applied |
| Incompatible endpoint version | Halt with the version gap named |

Filesystem events occurring while disconnected are lost, so **watch state is never trusted across a disconnect** — every reconnect performs a full rescan.

## Sync Fidelity and Attachment

An idle session otherwise holds an exec stream open through the runner's Kubernetes API server indefinitely. Because that cost scales with configured sessions rather than active engineers, sync fidelity follows shell attachment:

- **A shell session is attached** — the engineer is working. The stream stays open and changes propagate promptly in both directions.
- **No shell is attached** — the stream is dropped. Local watching continues and reconnects on local change or on a slow timer; changes originating in the sandbox surface on the next reconnect rather than immediately.

This reuses the attachment signal the Terminal Proxy already tracks and bounds steady-state cost to engineers actually working.

## Sync Engine

Reconciliation, the entry and snapshot model, conflict detection, ignore handling, symbolic-link modes, permission mapping, and staged transitions come from [Mutagen](https://github.com/mutagen-io/mutagen)'s `synchronization/core` package rather than being written from scratch. It is transport-agnostic — it operates on snapshots, not connections — which is what lets it sit behind an exec pipe it knows nothing about. The platform-specific work is the transport, the session semantics, and the safety model above.

Mutagen's own daemon, session manager, and agent-pushing machinery are **not** used. Those are what would put a second binary on each end and a version-negotiation protocol between them; the endpoint here is a subcommand of the CLI that is already present in every workload.

Three constraints on the dependency:

- **Only MIT-licensed packages.** The module is mixed-licensed: `sspl/` holds SSPL-covered enhancements (fanotify watching, zstd compression, xxh128 hashing) that official release builds include by default, and SSPL is written to reach exactly this platform's usage. Nothing under `sspl/` is used. The lost performance is recoverable — compression in particular belongs at the transport layer, which the platform owns.
- **Vendored, not a live module dependency.** The mixed license leaves the module classified as non-redistributable by Go tooling, so a live dependency would be flagged by every license scan and inventory. Vendoring the MIT packages with their notices preserved keeps the dependency graph honest and makes importing the wrong package structurally impossible rather than merely discouraged.
- **Delta transfer is separable.** Mutagen's `rsync` package is an in-process implementation of the rsync algorithm, not the `rsync` program, and it matters only for large files that change slightly. Whole-file transfer over a compressed stream is sufficient for working trees and is the starting point.

If the vendoring surface proves unworkable in practice, the fallback is a purpose-built engine over the same endpoint protocol. Nothing else in this document depends on which is used.

## Container-Side Constraints

| Constraint | Consequence |
|---|---|
| The endpoint shares the main container's cgroup | An over-eager scan can exhaust the container's memory and take the engineer's shell and running processes with it. Concurrency is bounded and content is streamed rather than buffered |
| `fs.inotify.max_user_watches` is a node-level limit | It cannot be raised from inside the container, and exhaustion is easy to swallow silently. The endpoint falls back to periodic scanning and **reports the degraded mode** rather than appearing to watch |
| Staging must share a filesystem with the root for atomic rename | It consumes the workspace volume; free space is checked before staging |
| The mounted root may already hold content | Content baked into the image, or left by earlier sessions. First contact with an unmarked non-empty root halts and requires a direction |
| The environment's workspace image defines the running user | Files are created as that user with default modes |

## Failure Modes

| Failure | Behavior |
|---|---|
| Endpoint terminated mid-transfer | Staged content is orphaned and inert; collected at the next handshake. Nothing partial is visible at its destination |
| Daemon crash | Sessions are on disk and resume on restart. An interrupted cycle recomputes |
| Machine sleeps | Streams die and are re-established on wake, with a full rescan |
| Sandbox idles out under an active session | Session pauses; the sandbox is not restarted |
| Sandbox terminated by TTL | Marker check halts the session; no local deletion occurs |
| Workspace volume full | Staging fails cleanly and the session reports the condition |
| CLI and in-sandbox versions diverge | Refused at handshake with an actionable message; no binary is pushed to close the gap |

## Related

- [Sandboxes](../product/sandboxes/sandboxes.md)
- [Terminal Proxy](terminal-proxy.md)
- [agyn-cli — Sandbox Sync Commands](agyn-cli.md#sandbox-sync-commands)
- [Runner — Execution](runner.md#execution)
- [Resource Definitions — Sandbox](resource-definitions.md#sandbox)
