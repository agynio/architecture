# Persistent Shells Are an Environment Setting

## Target

- [Resource Definitions — Environment](../architecture/resource-definitions.md#environment)
- [Runners — Workload Resource](../architecture/runners.md#workload-resource)
- [Agents Orchestrator — Workload Spec Assembly](../architecture/agents-orchestrator.md#workload-spec-assembly)
- [Terminal Proxy — Session Kinds](../architecture/terminal-proxy.md#session-kinds)
- [Terminal Proxy — Persistent Shells](../architecture/terminal-proxy.md#persistent-shells)
- [Sandboxes App](../architecture/sandboxes-app.md)
- [agyn-cli — Sandbox Commands](../architecture/agyn-cli.md#sandbox-commands)
- [Flavors and Environments](../product/environments/environments.md)
- [Sandboxes — Shell Access](../product/sandboxes/sandboxes.md#shell-access)

Supersedes the client-selected form introduced in [2026-08-11](2026-08-11-persistent-sandbox-shells.md). That change is not edited; this one states what replaces it.

## Delta

[Persistent shells](2026-08-11-persistent-sandbox-shells.md) arrived as a second session kind: a client asked for `SHELL_ATTACH` and got a shell that outlived its connection, or asked for `SHELL` and got one that did not. **That is the wrong place for the decision.**

Whether shells persist is a property of the runtime, not of the caller. It costs container memory, it changes what a stop destroys, and it should be the same for everyone working in one environment no matter which client they arrived through. A per-session choice means two clients against one sandbox can disagree, and the person who configured the environment has no say in either answer.

### The setting

`Environment` gains **`persistent_shells`**, defaulting to **true**. It governs every shell in every workload the environment runs.

Defaulting to on because it is the behavior people expect from a machine they connect to, and because the cost — a multiplexer holding PTYs and a bounded scrollback per shell — is small next to the workload it sits in. An environment turns it off when that memory is contended, or when its image supplies a multiplexer of its own that the platform's would sit inside.

### The kind goes away

`SESSION_KIND_SHELL_ATTACH` is removed. `SHELL` is the only interactive kind again, and it takes what naming a shell requires:

| Parameter | Meaning |
|---|---|
| `shell_id` | Which shell to attach to. Optional. Ignored where the environment does not persist shells |
| `shell_cwd` | Where a newly created shell starts. Optional, absolute |

A client no longer states what it wants; it states *which shell*, and the platform decides whether that shell survives the connection. A `SHELL` session with no `shell_id` is a shell nobody will return to — correct in both modes, and what the Console's agent-container terminal asks for.

**The parameters are not rejected when persistence is off.** A client that names a shell in an environment that does not keep them gets an ordinary ephemeral shell rather than an error: it asked for a thing the environment does not provide, which is the environment's answer to give, and failing the request would make every client branch on a setting it should not have to read.

### Where the proxy reads it

The workload record gains **`persistent_shells`**, resolved from the environment by the [Orchestrator](../architecture/agents-orchestrator.md) at start and recorded alongside the workload's other placement facts.

The Terminal Proxy already calls `Runners.GetWorkload` to find the hosting runner, so the answer arrives on a call it makes anyway. The alternative — the proxy resolving sandbox → environment through the Agents service — would add a dependency and a round trip to every ticket issuance, to read a value that cannot change for the life of the workload.

That it cannot change mid-workload is the point: an environment edited while a sandbox is running does not reconfigure that sandbox's shells underneath the person using them. The setting applies at the next start, like every other thing the environment decides.

## Acceptance Signal

- An engineer in a sandbox whose environment has `persistent_shells` on reloads the page and finds the same shell, with its work still running.
- The same sandbox in an environment with the setting off behaves as it did before persistence existed: a dropped connection ends the shell.
- Neither client asks for a kind. The Sandboxes app and `agyn sandbox connect` send `SHELL` with a `shell_id` in both cases, and behave correctly in both.
- A client naming a `shell_id` in a non-persistent environment gets a working ephemeral shell, not an error.
- Turning the setting off on an environment leaves a running sandbox's shells alone; a sandbox started afterwards has no persistent shells.
- A new environment created with no opinion gets persistent shells.

## Notes

- **The layout outlives the setting.** [Sandbox Layouts](../architecture/resource-definitions.md#sandbox-layout) are stored whether or not shells persist — tabs are a client's record of what it opened, and they are as useful for reopening ephemeral shells in the same directories as for returning to live ones.
- **Nothing about delivery changes.** `tmux` ships into every workload regardless, because the setting is resolved per workload at start and an image cannot be re-provisioned to acquire a multiplexer later.
- **No migration for existing environments.** The column defaults to true, which is the behavior this change introduces as standard.
