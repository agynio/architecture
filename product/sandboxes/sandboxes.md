# Sandboxes

## Purpose

A sandbox is an engineer-launched workload. Until now, workloads existed only when the [Agents Orchestrator](../../architecture/agents-orchestrator.md) started them for agent instances with unprocessed inbox items. Sandboxes let a person start a workload deliberately, get an interactive shell in it, and use it as a working environment with the platform's existing machinery — the [environment](../environments/environments.md)'s images and flavor, injected secrets, and egress network policy.

The primary use case is running agents in "manual mode": an engineer gets the same image, credentials, and network posture an agent workload would get, but drives it by hand — debugging tools, trying prompts, inspecting what an agent can and cannot reach. The API is designed so automations can start sandboxes later; v1 targets humans.

```bash
agyn sandbox start                 # start a sandbox, drop into a shell
# ... laptop sleeps, wifi drops ...
agyn sandbox connect brave-otter   # reattach to the same sandbox
```

## User Stories

- As an engineer, I want to start a sandbox from my terminal and get a shell in it, so I can work inside the same runtime my agents use.
- As an engineer, I want to reconnect to my sandbox by name after a disconnect, without losing running processes or files.
- As an engineer, I want the sandbox to carry the environment's secrets and egress rules, so I can manually exercise exactly what an agent in that environment could do.
- As an organization owner, I want to see every sandbox running in my organization and terminate any of them.
- As a platform operator, I want sandboxes to stop when idle and disappear after a TTL, so forgotten sandboxes don't consume capacity forever.

## Concepts

| Term | Definition |
|---|---|
| **Sandbox** | An org-scoped resource owned by the user who created it: one workload (plus a workspace volume) running an [environment](../environments/environments.md), started on demand rather than by inbox traffic. |
| **Shell session** | An interactive terminal attached to the sandbox's main container. A sandbox can have zero or more concurrent sessions; the workload keeps running between sessions. |

## CLI

Sandboxes are managed through a new `agyn sandbox` command group:

| Command | Description |
|---|---|
| `agyn sandbox start [--env NAME] [--name NAME]` | Create a sandbox, wait for the workload to run, attach a shell. `--env` selects the environment; defaults to the organization's sole environment when exactly one exists, otherwise required. `--name` sets the sandbox name; auto-generated (`adjective-noun`) when omitted |
| `agyn sandbox connect [NAME]` | Attach a shell to an existing sandbox. Calls `EnsureSandboxRunning` first: a no-op when `running`, a restart when `stopped`, a fresh start attempt when `failed` — the shell attaches only once the workload is running. With no argument: connects when the caller owns exactly one non-terminated sandbox, otherwise lists candidates |
| `agyn sandbox list [--all] [--terminated]` | List the caller's sandboxes: name, environment, status, age, last session. Terminated sandboxes are hidden unless `--terminated` is passed; `failed` ones are shown (they are actionable). `--all` lists every sandbox in the organization (owners) |
| `agyn sandbox stop [NAME]` | Stop the workload; keep the sandbox record and workspace volume |
| `agyn sandbox delete [NAME]` | Terminate the sandbox and delete its workspace volume |

Convenience: `agyn sandbox start --agent @coder` resolves the agent's environment. Note this grants the *environment's* attachments only — rules and ENVs attached directly to the agent do not apply to the sandbox.

## Lifecycle

| Status | Meaning |
|---|---|
| `starting` | Workload start requested, not yet running |
| `running` | Workload up; shell sessions can attach |
| `stopped` | Workload stopped (idle timeout or explicit `stop`); record and workspace volume retained |
| `failed` | Workload failed to start. Sticky until the user acts: `connect` performs a fresh start attempt (sandboxes have no background retry loop — nothing demands a sandbox run while nobody is connecting). TTL still applies |
| `terminated` | Deleted explicitly or by TTL; workspace volume removed. A soft state: the record is retained for audit and usage history but hidden from default lists |

- **Idle timeout** (default `30m`): measured from the last shell session detaching. When it elapses, the workload is stopped but the sandbox and its workspace volume survive — `connect` restarts the workload on the same volume. While a session is attached the sandbox is never considered idle.
- **TTL** (default `72h` from creation): when it elapses, the sandbox is terminated and its volume deleted regardless of state. The remaining TTL is visible in `agyn sandbox list` and the Console.
- **Defaults are organization-configurable** (org owners set default TTL and idle timeout in organization settings, within platform bounds). Both values are resolved and stored on the sandbox at creation — changing the org default later does not affect existing sandboxes.
- One workload at a time per sandbox. Sandboxes are first-class workload owners in the runtime model (`owner_kind=sandbox` on workload and volume records) — no agent instance exists behind a sandbox.

## What's Inside

A sandbox workload is assembled the same way an agent workload is, minus the agent loop:

- **Image and size** come from the environment (workspace image + flavor). Placement is on the environment's runner — see [Placement](../environments/environments.md#placement).
- **The platform's binary init containers run first**, so `agynd` and the `agyn` CLI are available inside the sandbox. The main container runs a long-lived sandbox holder process instead of the agent inbox loop. Session activity for idle tracking is reported by the [Terminal Proxy](../../architecture/terminal-proxy.md#sandbox-activity-reporting), not from inside the container.
- **The environment's agent runtime image, when it has one**, runs as a further init container, so the agent CLI it carries is on `PATH` inside the sandbox. This is what makes a sandbox a place to drive an agent by hand rather than an empty shell. The agent loop itself does not run — see [Future](#future).
- **Environment variables and secrets** attached to the environment are injected at workload assembly, exactly as for agents. Secret values are resolved by the orchestrator; they are never fetchable through the API from inside the sandbox.
- **Egress rules** attached to the environment apply: matched destinations route through the [Egress Gateway](../egress-gateway/egress-gateway.md) with the same allow/deny/inject behavior an agent observes.
- **Workspace volume**: a persistent volume mounted at `/workspace`, created with the sandbox and deleted with it. It survives idle stops and reconnects. Size is a platform default (`10Gi`, deployment-configurable) and the storage class is the runner's default — storage is not part of the flavor. It is a runtime-only volume owned by the sandbox, not a user-managed [Volume](../../architecture/resource-definitions.md#volume) resource.
- **Port exposure** works as in agent workloads: `agyn expose add 3000` inside the sandbox returns an `http://exposed-<id>.ziti:3000` URL reachable from the engineer's enrolled devices.
- **Identity and platform access**: the sandbox workload has its own platform identity (it is not the engineer's identity, and it is **not an organization member**) with the same platform service reach as an agent workload — Gateway, **LLM Proxy**, and Tracing over the standard OpenZiti dial policies. Reachability is not authorization: platform services authorize sandbox identities by resolving through the sandbox record to its organization and owner, granting only a narrow operation set (port exposure, file upload/download, LLM calls metered to the org and attributed to the sandbox and its owner). Model-calling agent tooling therefore works inside a sandbox without the sandbox holding any broad org permissions.

## Shell Access

`agyn sandbox start` and `connect` attach an interactive terminal to the sandbox's main container, streamed through the [Terminal Proxy](../../architecture/terminal-proxy.md) to the runner's `Exec` API. The Console offers the same terminal on the sandbox detail page.

The experience is SSH-parity — a real PTY with no platform-imposed limitations:

- Raw, 8-bit-clean byte stream: no line buffering, no filtering. Colors (through truecolor), alternate screen, mouse reporting, and full-screen TUIs (`vim`, `htop`, `tmux`) work.
- Terminal resize propagates (SIGWINCH), signals behave normally (`Ctrl-C`, job control), `isatty()` is true.
- No session wall timeout, idle timeout, or output cap. An attached session keeps the sandbox alive indefinitely; the sandbox `idle_timeout` counts only detached time.
- Multiple concurrent sessions per sandbox are allowed (same owner), each with its own PTY.
- The shell's exit code becomes the CLI's exit code.

A dropped connection ends the session but not the sandbox — like a dropped SSH connection, the foreground process group gets SIGHUP while the container keeps running. Anything that must survive a session drop should run under `nohup`/`tmux` (from the image); `agyn sandbox connect` opens a fresh shell. See [Terminal Proxy — Terminal Semantics](../../architecture/terminal-proxy.md#terminal-semantics).

## Permissions

| Action | Who |
|---|---|
| Create a sandbox | Any organization member |
| Connect (attach a shell) | The sandbox owner only |
| List | Owner sees own; organization owners (and cluster admins, as with workloads) see all |
| Stop / delete | The owner and organization owners |

Organization owners can force-terminate any sandbox but cannot attach to one they don't own.

Because environment-attached secrets and egress credentials are reachable from inside any sandbox running that environment, environments define the effective sharing boundary — see [Environments — Attachments](../environments/environments.md#attachments).

## Observability and Metering

- Sandboxes appear in the Console alongside agent workloads, marked as sandboxes, with owner, environment, status, and log/terminal access. Status changes are pushed live via `sandbox.updated` notifications — to the owner's room for their own list, and to an org-level room for the organization owners' list-all view.
- Egress from sandboxes emits the same spans as agent egress, attributed to the sandbox.
- Sandbox workloads emit the same `FLAVOR_SECONDS` metering records as agent workloads, attributed to the organization and labeled with the sandbox and its owner. A sandbox always has a flavor, since it always runs through an environment.

## Constraints

- A sandbox runs exactly one environment, fixed at creation. To use a different environment, start a new sandbox.
- No MCP sidecars or additional volumes in v1 — a sandbox is the environment's main container, its init containers, and platform sidecars.
- The environment's workspace image must provide a shell for sessions to be useful; the platform does not inject one.
- Shell access requires the Terminal Proxy path; there is no direct network path from the engineer's machine to the workload other than platform-mediated ones (terminal, port exposure).

## Future

- Automation-created sandboxes (CI jobs, scheduled maintenance) — the sandbox API is identity-agnostic by design.
- Running the actual agent loop inside a sandbox ("puppeteer an agent").
- Per-environment sandbox permission (`can_sandbox`) for organizations whose environments carry production credentials.
- Per-user/per-org sandbox quotas — v1 relies on idle timeout, TTL, and org-owner visibility.
- Mounting an agent instance's state volume into a sandbox for debugging (a data-access grant that needs its own design).
- Flavor-denominated metering — see [Environments — Metering](../environments/environments.md#metering).

## Related Architecture

- [Resource Definitions — Sandbox](../../architecture/resource-definitions.md#sandbox)
- [Environments](../environments/environments.md)
- [Agents Orchestrator](../../architecture/agents-orchestrator.md)
- [Terminal Proxy](../../architecture/terminal-proxy.md)
- [agyn-cli — Sandbox Commands](../../architecture/agyn-cli.md#sandbox-commands)
- [Expose Service](../../architecture/expose-service.md)
- [Secrets](../../architecture/secrets.md)
