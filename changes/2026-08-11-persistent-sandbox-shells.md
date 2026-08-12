# Persistent Sandbox Shells

## Target

- [Terminal Proxy — Session Kinds](../architecture/terminal-proxy.md#session-kinds)
- [Terminal Proxy — Terminal Semantics](../architecture/terminal-proxy.md#terminal-semantics)
- [Terminal Proxy — Wire Protocol](../architecture/terminal-proxy.md#wire-protocol)
- [Agent Init — Shared Volume Contract](../architecture/agent-init.md#shared-volume-contract)
- [agynd-cli — Shell Supervision](../architecture/agynd-cli.md#shell-supervision)
- [Resource Definitions — Sandbox Layout](../architecture/resource-definitions.md#sandbox-layout)
- [Agents Service — Sandbox Layout](../architecture/agents-service.md#sandbox-layout)
- [Agents Orchestrator — Layout Snapshot Before Stop](../architecture/agents-orchestrator.md#layout-snapshot-before-stop)
- [Notifications — Room Naming Convention](../architecture/notifications.md#room-naming-convention)
- [Sandboxes App](../architecture/sandboxes-app.md)
- [agyn-cli — Sandbox Commands](../architecture/agyn-cli.md#sandbox-commands)
- [Sandboxes — Shell Access](../product/sandboxes/sandboxes.md#shell-access)
- [Sandboxes App — Sandbox](../product/sandboxes/sandboxes-app.md#sandbox)

## Delta

**A shell lives and dies with the connection that opened it.** A dropped WebSocket ends the exec, the PTY closes, and the foreground process group receives SIGHUP. Reloading the Sandboxes app page discards every open tab; opening the sandbox on a second device shows none of them. The corpus states this as deliberate — server-side multiplexing was left to the image as a `tmux` concern.

It is the wrong side of the trade for the workload whose entire purpose is a person working inside it. [Sandboxes](../product/sandboxes/sandboxes.md#user-stories) already promises reconnecting "without losing running processes or files", and today that holds only for files and for work the engineer thought to background by hand.

This change makes a shell an object in the sandbox that outlives any connection to it, and makes the set of open shells state the platform stores rather than state a browser tab holds.

### Vocabulary

Three words that are currently one:

| Term | Meaning |
|---|---|
| **Shell** | The persistent thing — a PTY and its processes, living in the sandbox's main container, surviving every disconnect until the workload stops |
| **Terminal session** | The transport, unchanged: one ticket, one WebSocket, one `Runner.Exec`. A session **attaches to** a shell rather than being one |
| **Tab** | The Sandboxes app's presentation of a shell. UI vocabulary — the CLI has no tabs |

### tmux is the engine, not the interface

Shells are tmux sessions. A statically linked `tmux` ships with `agynd` in the [agynd-cli-init](../architecture/agent-init.md) image and lands on the shared volume; `agynd` starts the server at container boot on a platform-private socket with a platform-supplied configuration.

The platform's surface does not expose tmux. Clients name a shell by an opaque id through an ordinary [session kind](../architecture/terminal-proxy.md#session-kinds); the proxy binds a command that resolves to attach-or-create. Nothing above the container knows what implements a shell, which is what keeps the engine replaceable.

Building the multiplexer instead was considered and rejected. The only part that is genuinely hard is repainting a screen on attach — a raw byte replay cannot reconstruct a full-screen program, because a session in progress emits deltas against a base that has scrolled out of any buffer. Doing it correctly means maintaining a terminal emulator, which is the one part of tmux worth having and the one part expensive to own.

The cost accepted in exchange: **the byte stream is no longer uninterpreted.** [Terminal Semantics](../architecture/terminal-proxy.md#terminal-semantics) promised that anything the container emits and the client understands passes through untouched. tmux parses and re-emits, so what survives is what tmux knows. In practice this is invisible to the browser terminal, whose renderer is a narrower target than tmux's output; it is visible to a local terminal with capabilities tmux does not forward, and it is stated where the guarantee used to be.

### Two kinds, because two behaviors

`SHELL` keeps today's meaning and today's command: spawn a shell, and end it with the connection. A new `SHELL_ATTACH` binds the attach-or-create command and takes the shell id, plus an optional working directory used only when the shell is created.

Which one a consumer uses follows what the container is for. Inspecting an **agent** container stays ephemeral — the orchestrator may stop that workload mid-session and nothing there is worth preserving. Working in a **sandbox** is persistent, from the app and from the CLI both.

Attaching a second session to a live shell **displaces the first**. One shell has one attachment, which is what makes a page reload robust: the reloading client does not wait for its predecessor to be reaped, and no size negotiation between clients is needed because there are never two. The displaced client is told why it ended.

### The set of open shells is platform state

The tab list is a **layout document** stored by the Agents service, keyed by `(sandbox, identity)` and carrying a version for concurrent writers. It holds which tabs exist, their order, an optional name override, and each tab's last known working directory.

The container is not asked what shells exist. A tab is opened by attaching to its id, and attach-or-create means a tab whose tmux session is gone — because the sandbox restarted — simply comes back. Reconnecting and restoring are one path, not two.

Keyed by identity rather than by sandbox: a collaborator opening a shared sandbox gets their own tabs rather than joining the owner's. This is a presentation boundary and not a security one — anyone who can open a shell in the container can see every tmux session in it, as they can already read every file and signal every process.

**Working directories are captured before a planned stop.** A browser cannot be relied on at detach — a reload, a closed lid, and a dropped network all end the connection without giving the client a chance to write anything — and the value is only needed once the container is gone. So the [Orchestrator](../architecture/agents-orchestrator.md) snapshots the shells' directories into the layout immediately before it stops a workload, which covers idle timeout and explicit stop. A stop the platform did not initiate leaves the last captured value, and a tab opens where it last was rather than where it last looked.

### Restoring is honest about what it restores

A sandbox that stopped lost its processes. Tabs come back named and in the directory they were left in; they are **fresh shells**, and the app says so rather than presenting them as the ones that were there. This is the feature's own failure mode: persistence invites the belief that a build survives the night, and the idle timeout means it does not.

### Consequences that are not optional

**`PATH` moves into the tmux configuration.** The prefix is established today by the command the proxy binds, because that command *is* the shell. Under `SHELL_ATTACH` the bound command is a tmux client and the shell was spawned by the server, so the prefix has to be set where the server spawns shells. The same construction applies — the login profile runs first and `/agyn/bin` goes in front of what it left — only in a different place.

**Shells inherit `agynd`'s environment, not the exec's.** [agynd-cli](../architecture/agynd-cli.md#native-mode-configuration) reasons that an environment-variable placeholder must be on the container spec because a sandbox shell comes from `Runner.Exec` and inherits the spec rather than anything PID 1 assembled. The conclusion survives — a container-spec variable reaches `agynd` too — but the reason no longer describes what happens, and is corrected.

**The handshake carries `TERM`.** [The wire protocol](../architecture/terminal-proxy.md#wire-protocol) sends only the initial size, which was sufficient when the container's shell inherited whatever the pod spec had. A tmux client uses the attached terminal's `TERM` to decide what it may emit, truecolor included, so the client's value has to reach it. The field is optional and defaults to `xterm-256color`.

### CLI

`agyn sandbox connect` attaches to the caller's most recently attached shell and creates one when there are none, so a terminal and a browser reach the same work. `--new` opens another, `--tab` names one, and `agyn sandbox tabs` lists them. The CLI writes the layout document as the app does; a shell opened from a terminal is a tab in the browser.

## Acceptance Signal

- An engineer runs a build in a browser tab, reloads the page, and finds the same shell with the build still running.
- The same sandbox opened on a second device shows the same tabs; attaching to one displaces the first device, which says so rather than going silent.
- A full-screen program — `vim`, `htop` — is correct on reattach without a keystroke, at the new client's size.
- A shell opened with `agyn sandbox connect --new` appears as a tab in the browser, and a tab opened in the browser is what `agyn sandbox connect` returns to.
- A sandbox left to idle out and started again presents the same tabs, in the same order, in the directories they were left in, marked as fresh shells.
- Closing a tab ends its shell; `exit` inside a shell closes its tab.
- `agyn` and the environment's agent CLI are on `PATH` in a persistent shell, in an image whose `/etc/profile` sets `PATH` unconditionally.
- A terminal in an agent container behaves exactly as before and leaves nothing behind when it closes.
- A collaborator opening a shared sandbox starts with their own empty tab list, and the owner's tabs are unaffected by anything they do.

## Notes

- **No splits.** A tab is one shell is one tmux session with one window. Panes are the obvious way to grow into layout later and the model extends to them without breaking, but nothing here exposes them.
- **Two attachments to one shell** — genuine pairing, both parties watching the same screen — is not introduced. Displacement was chosen precisely to avoid negotiating one PTY size between clients, and reversing that is a deliberate later decision rather than a configuration change.
- **Shells do not survive the workload.** Nothing checkpoints a process; a stopped sandbox loses everything running in it. The layout is what survives, which is why restoring is described as recreation.
- **No client-facing shell listing.** The layout answers what tabs exist and the byte stream answers what each is called, so nothing the app draws requires asking the container. The one consumer that must — the Orchestrator, taking the directory snapshot — reaches the container over an internal path.
- **The Console's sandbox terminal is undecided here.** [Terminal Proxy](../architecture/terminal-proxy.md) names a terminal on the Console's Sandbox Detail page while [Sandboxes App](../product/sandboxes/sandboxes-app.md) states the Console deliberately has none. Pre-existing drift, untouched by this change; whichever way it resolves, persistence follows the sandbox rather than the client.
- **Idle timeout defaults are unchanged.** Raising them to protect long-running work is an organization policy question and belongs with the people who set the ceiling, not with this change.
