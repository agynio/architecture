# Sandboxes

## Target

- [Sandboxes](../product/sandboxes/sandboxes.md)
- [Resource Definitions — Sandbox](../architecture/resource-definitions.md#sandbox)
- [Terminal Proxy](../architecture/terminal-proxy.md)
- [Agents Orchestrator](../architecture/agents-orchestrator.md)
- [agyn-cli — Sandbox Commands](../architecture/agyn-cli.md#sandbox-commands)

## Delta

### Agents Service

- The Sandbox resource does not exist: no CRUD, no name generation, no per-org name uniqueness, no owner tracking, no status model, no org-configurable TTL default.

### Agents Orchestrator

- Sandbox reconciliation does not exist: start-on-create, stop on idle timeout (measured from last shell session detach), restart on connect, terminate on TTL, workspace volume lifecycle bound to the sandbox.
- Sandbox workloads (init image + long-lived holder process instead of the agent inbox loop) cannot be assembled. Sandbox workload identities are not created with the role attributes that grant Gateway/LLM Proxy/Tracing dial access or the `environment-<id>` egress attribute; LLM usage attribution to sandbox/owner does not exist.
- Sandbox workloads emit no metering records and there are no sandbox/owner labels.

### Terminal Proxy

- The Terminal Proxy is now specified ([terminal-proxy.md](../architecture/terminal-proxy.md)) but not implemented: no service, no `CreateTerminalSession` Gateway RPC, no ticket issuance, no WebSocket↔`Runner.Exec` bridging, no sandbox activity reporting (`TouchWorkload` heartbeats while sessions are attached, `last_session_at` on detach).
- `Runner.Exec` itself needs no changes: the proto already carries `ExecResize` (k8s-runner forwards it to the Kubernetes exec resize channel via `TerminalSizeQueue`) and unset wall/idle timeouts already mean unlimited (verified in `k8s-runner/internal/server/exec.go`). There is no initial-size field on `ExecStartRequest` — the proxy sends a resize message immediately after start.

### agynd

- No sandbox holder mode: `agynd` always runs the inbox loop. Sandbox workloads need a long-lived main process that does not consume an inbox.

### CLIs

- `agyn` has no `sandbox` command group (`start`/`connect`/`list`/`stop`/`delete`, `--env`/`--name`/`--agent` resolution, raw-mode TTY attach with resize/exit-code propagation over the Terminal Proxy).

### Authz

- No OpenFGA type/relations for sandboxes (member create, owner-only connect, owner/org-owner stop/delete, org-owner list-all).

### Console

- No sandbox list/detail (status, owner, TTL, terminal), no sandbox status-change notifications for live updates.

## Acceptance Signal

- An engineer runs `agyn sandbox start --env <name>`, gets a shell, disconnects, and `agyn sandbox connect <name>` reattaches with processes and `/workspace` intact — including after an idle stop.
- A secret-backed ENV and an egress rule attached to the sandbox's environment are observable from inside the sandbox (env var present; matched destination intercepted/injected).
- A sandbox left idle stops after its idle timeout; a sandbox past its TTL is terminated and its workspace volume deleted.
- An org owner sees all sandboxes via `agyn sandbox list --all` and can delete any; a non-owner member cannot connect to another member's sandbox.

## Notes

- Depends on [Flavors and Environments](2026-07-15-flavors-environments.md) — a sandbox specifies an environment; environment-targeted ENV, image pull secret, and egress attachments are delivered by that change.
- The Terminal Proxy spec ([terminal-proxy.md](../architecture/terminal-proxy.md)) was written as part of this change — it resolves a pre-existing dangling reference from Runners/OpenZiti docs. The Console workload terminal (agent containers, `can_edit_config`-gated) rides on the same service.
- Automation-created sandboxes are a design constraint (identity-agnostic API), not part of this delta.
