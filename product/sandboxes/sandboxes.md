# Sandboxes

## Purpose

A sandbox is an engineer-launched workload. Until now, workloads existed only when the [Agents Orchestrator](../../architecture/agents-orchestrator.md) started them for agent instances with unprocessed inbox items. Sandboxes let a person start a workload deliberately, get an interactive shell in it, and use it as a working environment with the platform's existing machinery — the [environment](../environments/environments.md)'s images and flavor, injected secrets, and egress network policy.

The primary use case is running agents in "manual mode": an engineer gets the same image, credentials, and network posture an agent workload would get, but drives it by hand — debugging tools, trying prompts, inspecting what an agent can and cannot reach. The API is designed so automations can start sandboxes later; v1 targets humans.

```bash
agyn sandbox start                 # start a sandbox, drop into a shell
# ... laptop sleeps, wifi drops ...
agyn sandbox connect brave-otter   # reattach to the same sandbox

agyn sandbox sync                  # keep the working directory and /workspace in step
```

## User Stories

- As an engineer, I want to start a sandbox from my terminal and get a shell in it, so I can work inside the same runtime my agents use.
- As an engineer, I want to reconnect to my sandbox by name after a disconnect, without losing running processes or files.
- As an engineer, I want a directory on my machine and a directory in the sandbox to stay in step, so I can edit in my own editor while the work runs in the sandbox and see whatever it writes back.
- As an engineer, I want the sandbox to carry the environment's secrets, egress rules, MCP servers, and storage layout, so I can manually exercise exactly what an agent in that environment could do.
- As an engineer, I want to know before I start working whether my files will survive the sandbox going idle.
- As an engineer, I want to say how long my sandbox should outlive my last connection — a four-hour build and a five-minute experiment do not want the same answer — and to set my own usual value once rather than on every start.
- As an organization owner, I want a ceiling on that, so a forgotten sandbox on expensive hardware cannot be pinned alive by whoever started it.
- As an engineer, I want to give a colleague a shell in the sandbox I am debugging, without giving them anything else, and to take it back when we are done.
- As an engineer, I want a sandbox someone shared with me to show up in my own list, and to be able to restart it when it has idled out.
- As an organization owner, I want to see every sandbox running in my organization and terminate any of them.
- As a platform operator, I want sandboxes to stop when idle and disappear after a TTL, so forgotten sandboxes don't consume capacity forever.

## Concepts

| Term | Definition |
|---|---|
| **Sandbox** | An org-scoped resource owned by the user who created it: one workload running an [environment](../environments/environments.md), started on demand rather than by inbox traffic. Its storage is whatever the environment declares. |
| **Shell session** | An interactive terminal attached to the sandbox's main container. A sandbox can have zero or more concurrent sessions; the workload keeps running between sessions. |
| **Sync session** | A continuous two-way reconciliation between a directory on the engineer's machine and a directory in the sandbox. Runs in the background on the engineer's machine, independent of any shell session. |

## CLI

Sandboxes are managed through a new `agyn sandbox` command group:

| Command | Description |
|---|---|
| `agyn sandbox start [--env NAME] [--name NAME] [--sync PATH] [--idle-timeout DURATION]` | Create a sandbox, wait for the workload to run, attach a shell. `--env` selects the environment; defaults to the organization's sole environment when exactly one exists, otherwise required. `--name` sets the sandbox name; auto-generated (`adjective-noun`) when omitted. `--idle-timeout` overrides how long the sandbox survives unattended — see [Choosing an idle timeout](#choosing-an-idle-timeout) |
| `agyn sandbox connect [NAME]` | Attach a shell to an existing sandbox. Calls `EnsureSandboxRunning` first: a no-op when `running`, a restart when `stopped`, a fresh start attempt when `failed` — the shell attaches only once the workload is running. With no argument: connects when the caller owns exactly one non-terminated sandbox, otherwise lists candidates |
| `agyn sandbox list [--all] [--terminated]` | List the caller's sandboxes: name, environment, status, age, last session, idle timeout, remaining TTL. Terminated sandboxes are hidden unless `--terminated` is passed; `failed` ones are shown (they are actionable). `--all` lists every sandbox in the organization (owners) |
| `agyn sandbox stop [NAME]` | Stop the workload; keep the sandbox record and its persistent volumes. Warns first when the environment declares none — stopping discards everything |
| `agyn sandbox delete [NAME]` | Terminate the sandbox and delete the volumes provisioned for it |
| `agyn sandbox cp [-r] SRC DST` | Copy files in or out once. The sandbox side carries a `NAME:path` prefix |
| `agyn sandbox sync [NAME] [--local PATH] [--remote PATH]` | Keep a local directory (default the working directory) and a sandbox directory (default `/workspace`) continuously reconciled. See [Workspace Sync](#workspace-sync) |

`--remote` defaults to `/workspace` because that is the conventional mount path, not because the platform creates one. In an environment that mounts its volume elsewhere, pass the path.

Convenience: `agyn sandbox start --agent @coder` resolves the agent's environment. Note this grants the *environment's* attachments only — rules and ENVs attached directly to the agent do not apply to the sandbox. `agyn sandbox start --sync .` starts a sandbox, attaches a shell, and begins syncing the current directory in one step.

## Lifecycle

| Status | Meaning |
|---|---|
| `starting` | Workload start requested, not yet running |
| `running` | Workload up; shell sessions can attach |
| `stopped` | Workload stopped (idle timeout or explicit `stop`); record and any persistent volumes retained |
| `failed` | Workload failed to start. Sticky until the user acts: `connect` performs a fresh start attempt (sandboxes have no background retry loop — nothing demands a sandbox run while nobody is connecting). TTL still applies |
| `terminated` | Deleted explicitly or by TTL; provisioned volumes removed. A soft state: the record is retained for audit and usage history but hidden from default lists |

- **Idle timeout** (default `30m`, set per sandbox at creation): idle means **nothing is attached**. While a session is attached the sandbox is never considered idle; the clock starts at the last detach. When it elapses, the workload is stopped; the sandbox record and the environment's persistent volumes survive, and `connect` restarts the workload on the same disks. **Anything outside a persistent volume does not survive** — see [Storage](#storage). A [sync session](#workspace-sync) is deliberately **not** an attachment for this purpose: a laptop left syncing would otherwise keep a sandbox running indefinitely. See [Choosing an idle timeout](#choosing-an-idle-timeout).
- **TTL** (default `72h` from creation): when it elapses, the sandbox is terminated and its volumes deleted regardless of state. The remaining TTL is visible in `agyn sandbox list` and the Console.
- **Defaults are organization-configurable** (org owners set the default TTL, the default idle timeout, and the maximum idle timeout a creator may request, within platform bounds). Values are resolved and stored on the sandbox at creation — changing an org setting later does not affect existing sandboxes.

### Choosing an idle timeout

Thirty minutes fits an engineer stepping away from a shell. It fits nothing else: a long build, a call, a run left going overnight all want a different number, and the person starting the sandbox knows which.

```bash
agyn sandbox start --env gpu --idle-timeout 4h
agyn profile set cloud --sandbox-idle-timeout 2h    # my default, this profile
```

The value resolves flag → local profile default → organization default. An organization owner sets the ceiling (`sandbox_max_idle_timeout`); a request above it is **rejected, naming the ceiling**, rather than quietly reduced to a number the engineer never sees and would plan around wrongly. TTL is the backstop underneath: a long idle timeout keeps a sandbox alive while nobody is attached, but nothing outlives its TTL.

Two bounds govern a sandbox's life, so `agyn sandbox list` shows both — the idle timeout and the remaining TTL — rather than making one of them something you go looking for.
- One workload at a time per sandbox. Sandboxes are first-class workload owners in the runtime model (`owner_kind=sandbox` on workload and volume records) — no agent instance exists behind a sandbox.

## What's Inside

A sandbox workload is assembled the same way an agent workload is, minus the agent loop:

- **Image and size** come from the environment (workspace image + flavor). Placement is on the environment's runner — see [Placement](../environments/environments.md#placement).
- **The platform's binary init containers run first**, so `agynd` and the `agyn` CLI are available inside the sandbox. The main container runs `agynd` as a holder: it runs the environment's init scripts and then idles, spawning no agent CLI and running no inbox loop. Session activity for idle tracking is reported by the [Terminal Proxy](../../architecture/terminal-proxy.md#sandbox-activity-reporting), not from inside the container.
- **The environment's agent runtime image, when it has one**, runs as a further init container, so the agent CLI it carries is on `PATH` inside the sandbox. This is what makes a sandbox a place to drive an agent by hand rather than an empty shell. The agent loop itself does not run — see [Future](#future).
- **The agent CLI can call models, without anyone configuring it.** The environment's [LLM access mode](../environments/environments.md#llm-access) applies to a sandbox exactly as it does to an agent: in `platform` mode the CLI is pointed at the platform's LLM endpoint; in `native` mode it runs in its stock configuration against the environment's subscription, with the credential held by the platform rather than placed in the container. A person at the shell types `claude` and it works, using the same models and the same accounting an agent in that environment would. In `native` mode the CLI keeps its own model picker — the platform pins nothing for a sandbox, because the person driving it is right there.
- **Environment variables and secrets** attached to the environment are injected at workload assembly, exactly as for agents. Secret values are resolved by the orchestrator; they are never fetchable through the API from inside the sandbox.
- **Egress rules** attached to the environment apply: matched destinations route through the [Egress Gateway](../egress-gateway/egress-gateway.md) with the same allow/deny/inject behavior an agent observes.
- **The environment's [volumes](../../architecture/resource-definitions.md#volume)**, provisioned per sandbox and mounted where the environment declares them. Persistent ones survive idle stops and reconnects; they are deleted when the sandbox is. **An environment declaring no persistent volume gives the sandbox nothing that outlives its workload** — see [Storage](#storage).
- **The environment's MCP servers**, running as sidecars exactly as they would in an agent workload. The agent loop is absent; its tools are not.
- **The environment's init scripts**, executed before the shell is available.
- **Port exposure** works as in agent workloads: `agyn expose add 3000` inside a sandbox named `super-sandbox` returns `http://super-sandbox.acme.agyn:3000`, reachable from the engineer's enrolled devices. The address is the sandbox's own — it survives an idle stop and a restart, so it can be bookmarked. The [Sandboxes app](sandboxes-app.md#ports) does the same from the sandbox page, for anyone who could open a shell in it. See [Port Exposure — Link Format](../port-exposure/port-exposure.md#link-format).
- **Identity and platform access**: the sandbox workload has its own platform identity (it is not the engineer's identity, and it is **not an organization member**) with the same platform service reach as an agent workload — Gateway, **LLM Proxy**, and Tracing over the standard OpenZiti dial policies. Reachability is not authorization: platform services authorize sandbox identities by resolving through the sandbox record to its organization and owner, granting only a narrow operation set (port exposure, file upload/download, LLM calls metered to the org and attributed to the sandbox and its owner). Model-calling agent tooling therefore works inside a sandbox without the sandbox holding any broad org permissions.

Starting a sandbox in an environment requires `can_use` on it — see [Who Can Use an Environment](../environments/environments.md#who-can-use-an-environment). A shell in a sandbox reaches the environment's secrets and credentials, so the right to open one is granted deliberately.

## Storage

**A sandbox has no storage of its own.** It mounts the [volumes](../environments/environments.md#volumes) its environment declares, one set per sandbox, and nothing else. There is no implicit `/workspace` disk: an environment that declares a persistent volume at `/workspace` gives its sandboxes a `/workspace` that survives an idle stop, and an environment that declares nothing gives them a container filesystem that is discarded when the workload stops.

This is the same rule agents follow, and it is deliberate — persistence is something an operator asks for per environment, not something every workload gets by default. What it means for an engineer at a shell:

| Environment declares | Sandbox stops (idle or explicit) | Sandbox terminated (TTL or `delete`) |
|---|---|---|
| A persistent volume | Files under it survive; `connect` returns to them | Deleted with the sandbox |
| Nothing persistent | Everything is lost | Nothing to delete |

Because the second row is easy to walk into, `agyn sandbox start` and the Console say so up front when the chosen environment declares no persistent volume, and `agyn sandbox stop` warns before stopping such a sandbox. The condition is visible in `agyn sandbox list` and on the sandbox detail page, not only at the moment work is lost.

Two sandboxes in the same environment never share storage — same layout, separate disks — and neither do a sandbox and an agent running there. [Workspace sync](#workspace-sync) is the way work reaches an engineer's machine, and in an environment with no persistent volume it is also the only thing standing between an idle timeout and a lost afternoon.

## Shell Access

`agyn sandbox start` and `connect` attach an interactive terminal to the sandbox's main container, streamed through the [Terminal Proxy](../../architecture/terminal-proxy.md) to the runner's `Exec` API. The [Sandboxes app](sandboxes-app.md) offers the same terminal in the browser, as does the Console on its sandbox detail page.

The experience is SSH-parity — a real PTY with no platform-imposed limitations:

- Raw, 8-bit-clean byte stream: no line buffering, no filtering. Colors (through truecolor), alternate screen, mouse reporting, and full-screen TUIs (`vim`, `htop`, `tmux`) work.
- Terminal resize propagates (SIGWINCH), signals behave normally (`Ctrl-C`, job control), `isatty()` is true.
- No session wall timeout, idle timeout, or output cap. An attached session keeps the sandbox alive indefinitely; the sandbox `idle_timeout` counts only detached time.
- Multiple concurrent sessions per sandbox are allowed, each with its own PTY — several from the owner, or the owner and a [collaborator](#sharing) working side by side.
- The shell's exit code becomes the CLI's exit code.

A dropped connection ends the session but not the sandbox — like a dropped SSH connection, the foreground process group gets SIGHUP while the container keeps running. Anything that must survive a session drop should run under `nohup`/`tmux` (from the image); `agyn sandbox connect` opens a fresh shell. See [Terminal Proxy — Terminal Semantics](../../architecture/terminal-proxy.md#terminal-semantics).

## Workspace Sync

A sandbox is only as useful as the files in it. Workspace sync keeps a directory on the engineer's machine and a directory in the sandbox continuously reconciled in both directions, so work can be edited locally with local tooling and run inside the sandbox, and whatever the sandbox writes appears back on the machine.

Both sides remain ordinary local filesystems — nothing is mounted over the network. Editors, watchers, builds, and `git` run at local disk speed on both ends, and a sleeping laptop or a dropped connection stalls nothing locally.

```bash
agyn sandbox sync                     # sync the working directory; returns immediately
agyn sandbox sync status              # what is syncing, and anything needing attention
agyn sandbox sync stop api-brave-otter
```

For a single transfer rather than an ongoing relationship, `agyn sandbox cp` copies files in or out once and exits.

- **It runs in the background.** Sync outlives the command that started it and is unaffected by closing the terminal. It does not start unless asked: only `agyn sandbox sync` and an explicit daemon start bring it up, and it exits when the last session is removed. Nothing resumes automatically after a reboot — sessions are listed as not running until started again, or an opt-in login item is installed.
- **It never interrupts.** Sync does not prompt. When something needs a decision it stops making that particular change and reports, and the engineer resolves it whenever they get to it — inline if a session is attached, in a status file inside the synced directory, as a desktop notification, through `agyn sandbox sync status`, and as a banner on the next `agyn sandbox connect`.
- **Conflicts are per-file.** When both sides change the same file, that path is set aside with both versions intact and everything else keeps syncing.
- **Deletions are recoverable.** Files removed locally by sync go to a retained trash directory rather than being unlinked, so a surprising deletion can be undone instead of having to be prevented.
- **A destroyed sandbox never destroys local work.** A sandbox terminated by its TTL comes back with an empty workspace. Sync recognizes this and stops rather than mirroring the emptiness onto the engineer's machine.
- **A stopped sandbox pauses sync; it does not restart it.** Editing a local file must never silently start billable compute. Sync resumes on the next `agyn sandbox connect`.
- **Ownership is not carried across.** Only the executable bit is preserved; files are created as the sandbox image's user.

Sync is intended for working trees — source, configuration, notes, results. It is not a data-transfer mechanism for large datasets: those belong on the sandbox side, reached through the environment or the organization's own storage.

## Sharing

A sandbox owner can give other identities in their organization access to that one sandbox — the colleague who should look at the failing build, the teammate taking over a long-running job.

**What a collaborator can do:** open a shell (and a sync session, which reaches the same filesystem), start the sandbox, and stop it. Starting matters: a shared sandbox that idled out overnight is useless to someone who cannot bring it back.

**What a collaborator cannot do:** delete the sandbox, or share it with a third person. The share list belongs to the owner.

**A share does not carry across to the environment.** Being able to start sandboxes in an environment is a separate permission held by separate people; a collaborator cannot start a second sandbox in the environment behind the one they were given.

What a share *does* hand over is everything reachable from a shell in that sandbox: the environment's secret-backed ENVs, the credentials its egress rules inject, and the contents of its volumes. The person receiving it may hold no role on that environment at all. This is the intended behavior — the owner is trusted to decide who joins a sandbox they started — and it is the reason the act of sharing states its own consequences rather than reading as an invitation.

Two further consequences worth knowing before sharing:

- **Compute a collaborator starts bills to the owner.** Usage stays attributed to the sandbox and its owner no matter who pressed start.
- **Unsharing takes effect on the next connection.** A session already open runs until it ends, like an SSH connection whose key was removed mid-session.

Shares live on the sandbox and disappear with it: a deleted sandbox and its replacement share nothing, including their collaborators.

## Permissions

| Action | Who |
|---|---|
| Create a sandbox | Any organization member, with `can_use` on the environment |
| Connect (attach a shell) | The sandbox owner, and the identities the owner has [shared](#sharing) it with |
| Sync a workspace | Same as attaching a shell — a sync session reaches the same filesystem |
| List | Owner sees own and shared-with-them; organization owners (and cluster admins, as with workloads) see all |
| Start / stop | The owner, collaborators, and organization owners |
| Delete | The owner and organization owners |
| Share / unshare | The owner only |

Organization owners can force-terminate any sandbox but cannot attach to one on that basis — entering someone else's sandbox requires a share, the same as for anyone else.

**Two permissions, two questions.** `can_use` on an environment answers *may this person start a sandbox here* — see [Environments — Who Can Use an Environment](../environments/environments.md#who-can-use-an-environment). A share answers *may this person enter this particular sandbox*. They are independent: a collaborator needs only the second, and holding it grants nothing about the environment behind the sandbox.

That independence is what makes sharing worth stating carefully, because environment-attached secrets and egress credentials are reachable from inside any sandbox running that environment — see [Sharing](#sharing).

## Observability and Metering

- The [Sandboxes app](sandboxes-app.md) is where a member sees their own sandboxes and the ones shared with them. Sandboxes also appear in the Console alongside agent workloads, marked as sandboxes, with owner, environment, status, and log/terminal access — that view is organization-wide and exists for owners overseeing the fleet.
- Status changes are pushed live via `sandbox.updated` notifications — to the owner's room for their own list, to a per-sandbox room that also reaches collaborators, and to an org-level room for the organization owners' list-all view.
- Egress from sandboxes emits the same spans as agent egress, attributed to the sandbox.
- Sandbox workloads emit the same `FLAVOR_SECONDS` metering records as agent workloads, attributed to the organization and labeled with the sandbox and its owner. A sandbox always has a flavor, since it always runs through an environment.

## Constraints

- A sandbox runs exactly one environment, fixed at creation. To use a different environment, start a new sandbox.
- No MCP sidecars or additional volumes in v1 — a sandbox is the environment's main container, its init containers, and platform sidecars.
- The environment's workspace image must provide a shell for sessions to be useful; the platform does not inject one.
- Shell access requires the Terminal Proxy path; there is no direct network path from the engineer's machine to the workload other than platform-mediated ones (terminal, port exposure, sync).
- Sync propagates changes; it does not mount. The two directories are separate filesystems kept in step, so a change is visible on the other side shortly after it is made, not at the instant of the write.
- While no shell session is attached, sync reduces to periodic reconciliation — changes originating in the sandbox surface on the next pass rather than immediately.

## Future

- Automation-created sandboxes (CI jobs, scheduled maintenance) — the sandbox API is identity-agnostic by design.
- Running the actual agent loop inside a sandbox ("puppeteer an agent").
- Per-environment sandbox permission (`can_sandbox`) for organizations whose environments carry production credentials.
- Per-user/per-org sandbox quotas — v1 relies on idle timeout, TTL, and org-owner visibility.
- Organization policy on sharing — confining shares to identities that already hold `can_use` on the sandbox's environment. v1 leaves the decision with the sandbox owner.
- `agyn sandbox share` — sharing is API-backed and reachable from the [Sandboxes app](sandboxes-app.md); the CLI counterpart is not yet specified.
- Mounting an agent instance's state volume into a sandbox for debugging (a data-access grant that needs its own design).
- Flavor-denominated metering — see [Environments — Metering](../environments/environments.md#metering).

## Related Architecture

- [Sandboxes App](sandboxes-app.md) — the browser surface for the above
- [Resource Definitions — Sandbox](../../architecture/resource-definitions.md#sandbox)
- [Environments](../environments/environments.md)
- [Agents Orchestrator](../../architecture/agents-orchestrator.md)
- [Terminal Proxy](../../architecture/terminal-proxy.md)
- [Sandbox Workspace Sync](../../architecture/sandbox-sync.md)
- [agyn-cli — Sandbox Commands](../../architecture/agyn-cli.md#sandbox-commands)
- [Expose Service](../../architecture/expose-service.md)
- [Secrets](../../architecture/secrets.md)
