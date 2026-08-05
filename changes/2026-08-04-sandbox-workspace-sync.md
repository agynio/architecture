# Sandbox Workspace Sync

## Target

- [Sandbox Workspace Sync](../architecture/sandbox-sync.md)
- [Sandboxes — Workspace Sync](../product/sandboxes/sandboxes.md#workspace-sync)
- [Terminal Proxy — Session Kinds](../architecture/terminal-proxy.md#session-kinds)
- [Gateway — Terminal Sessions](../architecture/gateway.md#terminal-sessions)
- [agyn-cli — Sandbox Sync Commands](../architecture/agyn-cli.md#sandbox-sync-commands)

## Delta

### Gateway

- `CreateTerminalSession` takes a free-form optional command override rather than a **session kind**. The kind-to-command mapping, the `SYNC` kind, and root-path validation and normalization for it do not exist. Tickets are not bound to a kind.

### Terminal Proxy

- Sessions are TTY-only. Non-TTY session kinds — no PTY, no resize — are not supported.
- Both `Runner.Exec` output streams are written to the WebSocket as untagged binary frames. That is correct for TTY sessions, where a PTY merges them anyway, but leaves a non-TTY consumer unable to separate protocol from diagnostics — a single stderr byte corrupts the stream. Non-TTY kinds need `separate_stderr` requested and each binary frame tagged with its source stream.
- `SESSION_KIND_UNSPECIFIED` has no defined rejection behavior, and there is no kind-to-command table, no absolute-path invocation for the in-sandbox binary, and no root-path validation or workspace confinement.
- Sandbox activity reporting does not discriminate by kind: every attached session drives `TouchWorkload` and updates `last_session_at`. `SYNC` sessions must do neither, or a background sync session keeps a sandbox running for as long as an engineer's machine stays connected.

### agyn (in-sandbox)

- No `sandbox sync serve` endpoint: no root marker and identity reporting, no protocol-version negotiation, no scan with a supplied digest cache, no blocking change poll with degraded-watch reporting, no content staging on the workspace filesystem, no atomic transition with per-path results, no collection of staging left by terminated sessions.
- `agyn` is not on `PATH` for an exec'd process — `PATH` is extended only by `agynd` when it spawns the agent subprocess, and a sandbox does not run `agynd`. Sync invokes it by absolute path and needs no `PATH` at all; the interactive case is covered by [Agent Volume Layout](2026-08-04-agyn-volume-layout.md).

### agyn CLI

- No `sandbox cp`: no one-shot transfer in either direction, no `NAME:path` addressing of the sandbox side.
- No `sandbox sync` command group: no session create/list/status/pause/resume/stop, no conflict resolution, no halted-session reset, no undelete from trash, no `--foreground`, no `--sync` on `sandbox start`.
- No resolution of a sandbox from the working directory when that directory already belongs to a session.
- No local sync daemon: no detached lifecycle independent of the invoking terminal, no owner-only socket and single-instance guard, no persisted sessions with a reconciliation base and trash, no version-mismatch restart, no opt-in resume-at-login service.
- No sync engine. Reconciliation, conflict detection, ignore handling, symbolic-link and permission modes, and staged transitions are not present in the CLI in any form.
- No reconnection model: no classification of transport failure, sandbox stop, sandbox termination, and root-identity mismatch into distinct outcomes, and no full rescan on reconnect.
- No halt reporting: no sentinel file in the sync root, no desktop notification, no non-zero status exit, no banner on `sandbox connect`.

### Runners

- No change. `Runner.Exec` already supports non-interactive mode with stdout and stderr separated.

### Authorization

- No change. Sync is authorized as sandbox `owner`, the check `CreateTerminalSession` already performs for shell sessions.

### Console

- No change, and none planned. Sync is a CLI-only surface; the Console shows sandboxes as it does today.

## Acceptance Signal

- An engineer runs `agyn sandbox sync` in a project directory, gets their prompt back, closes the terminal, and edits a file locally — the change appears in the sandbox. A file written inside the sandbox appears locally.
- `agyn sandbox cp` moves a file in and back out again without a daemon or a session being created.
- Changing the same file on both sides quarantines that path with both versions intact while every other path continues to sync; `agyn sandbox sync resolve` clears it.
- A sandbox terminated by its TTL leaves the local directory untouched: the session halts on the root-identity check and requires `sync reset` to continue.
- A sandbox left idle under an active sync session stops on schedule, and the session pauses without restarting it. `agyn sandbox connect` resumes both.
- A file deleted locally by sync is recoverable with `agyn sandbox sync undelete`.
- Running any other `agyn` command does not start the sync daemon. After a reboot, `agyn sandbox sync list` reports sessions as not running rather than appearing to sync.
- The sandbox's workspace and the local root disagree on nothing after a disconnect long enough to lose filesystem events on both sides.

## Notes

- Depends on [Sandboxes](2026-07-15-sandboxes.md) — sync targets a sandbox's workspace and reuses the Terminal Proxy session path that change introduces.
- Assumes the `agyn` CLI is present in every sandbox, which the platform's binary init containers provide. Sync ships no binary of its own to either end.
- Which reconciliation engine is vendored is unsettled, under the constraints in [Sync Engine](../architecture/sandbox-sync.md#sync-engine). The leading candidate's core packages are permissively licensed and transport-agnostic, but the module carries a mixed license overall and its transitive surface is unmeasured. The alternative is a purpose-built engine. Nothing else in this delta depends on which is chosen.
- Delta transfer for large, slightly changed files is out of scope; whole-file transfer over a compressed stream is sufficient for working trees.
- Shared read-write-many volumes across several workloads are a separate capability and not part of this delta. They would change what sync targets, not how it works.
