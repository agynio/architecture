# Agent Volume Layout

## Target

- [Agent Init Container — Shared Volume Contract](../architecture/agent-init.md#shared-volume-contract)
- [Agent Init Container — config.json](../architecture/agent-init.md#configjson)
- [agynd-cli — Agent Subprocess](../architecture/agynd-cli.md)
- [Agents Orchestrator](../architecture/agents-orchestrator.md)
- [Terminal Proxy — Session Kinds](../architecture/terminal-proxy.md#session-kinds)

## Delta

The shared volume is mounted at `/agyn-bin` with binaries at two levels — `agynd` and the agent CLI at the root, `agyn` under `cli/` — and `config.json` sitting among them. Two `PATH` entries are needed to cover both levels, which puts a non-binary on `PATH`, and the mount's name describes only part of what it holds.

The target is one mount at `/agyn`, every binary in `/agyn/bin`, and configuration beside it:

```
/agyn/
├── bin/{agynd, agyn, <agent-cli>}
└── config.json
```

### Platform binary init images

- `agynd-cli-init` writes `agynd` to the volume root; it must write to `bin/`.
- `agyn-cli-init` writes `agyn` to `cli/`; it must write to `bin/`.

### Agent runtime images

- Runtime images write the agent CLI to the volume root and `config.json` beside it. The CLI must go to `bin/` and `config.json` to the volume root.
- `config.json`'s `bin` field is an absolute path into the platform's mount (`/agyn-bin/codex`). It must become **relative to the volume** (`codex`), so an image states what it carries rather than asserting where the platform mounted it. Every published runtime image is rebuilt; no compatibility path is kept.
- `agyn` and `agynd` are not reserved names in the shared directory. They must be, now that all three writers share `bin/`.

### agynd

- Prepends two `PATH` entries (`/agyn-bin/cli` and `/agyn-bin`) when spawning the agent subprocess; one (`/agyn/bin`) is enough and keeps `config.json` off `PATH`.
- Resolves `config.json`'s `bin` as an absolute path. It must resolve it against the volume root and fail loudly on an absolute value rather than accepting both forms.

### Agents Orchestrator

- Workload assembly names the volume `agyn-bin`, mounts it at `/agyn-bin`, and sets the main container command to `/agyn-bin/agynd`. All three follow the new layout.

### Terminal Proxy

- The proxy is implemented and issues shell sessions today, with the command hardcoded as `exec ${SHELL:-sh} -l` in the service and mirrored in its ticket store. That form both omits the platform binaries from `PATH` and, being a login shell, would discard a `PATH` set ahead of it. It must become the profile-first form in [Terminal Proxy — PATH in a session](../architecture/terminal-proxy.md#path-in-a-session), and the command must stop being a hardcoded constant now that it varies by session kind.

### k8s-runner

- No change. The mount path is workload-spec data; the runner does not interpret it.

## Acceptance Signal

- An agent workload starts with one `emptyDir` mounted at `/agyn`, holding `bin/agynd`, `bin/agyn`, the environment's agent CLI in `bin/`, and `config.json` at the root.
- `which agyn` and `which <agent-cli>` both resolve inside the agent subprocess with a single `/agyn/bin` entry on `PATH`, and `config.json` is not on `PATH`.
- A runtime image whose `config.json` carries `"bin": "codex"` runs; one carrying an absolute path fails at startup with a message naming the field.
- A sandbox shell on a Debian-family workspace image has `agyn` and the environment's agent CLI on `PATH`, ahead of everything `/etc/profile` set — the case a `PATH` assigned before a login shell would have lost.
- A non-root workspace image runs unchanged — the volume is read by the main container, so no ownership fixing is introduced.

## Notes

- Splitting `agyn` into `cli/` originally kept `PATH` pointed at a directory holding only the platform CLI. That rationale ended when `/agyn-bin` itself was added to `PATH` so the agent CLI could be found by name; the split has been vestigial since, and the layout was never revisited.
- `/agyn` was previously a **writable** mount created by a permission-fixing init container, removed by [Home and Workspace Env](2026-04-24-home-workspace-env.md). This change reuses the path for a read-only delivery volume — no directory pre-creation, no ownership fixing, and no injected `HOME` or `WORKSPACE_DIR`.
- Every published agent runtime image must be rebuilt. This is accepted rather than mitigated: a compatibility path would preserve the absolute-path coupling that motivates the change.
- [Sandbox Workspace Sync](2026-08-04-sandbox-workspace-sync.md) depends on the `PATH` behavior described here for interactive sessions. Its own endpoint is invoked by absolute path and is unaffected by the layout.
