# HOME and WORKSPACE_DIR Not Platform-Managed

## Target

- [Agents Orchestrator — Workload Spec Assembly](../architecture/agents-orchestrator.md#workload-spec-assembly)
- [agynd — Environment Preparation](../architecture/agynd-cli.md#3-environment-preparation)
- [agynd — LLM Endpoint Configuration](../architecture/agynd-cli.md#llm-endpoint-configuration)

## Delta

The Agents Orchestrator currently injects `HOME=/tmp` and `WORKSPACE_DIR=/workspace` into the agent container, treats them as reserved env vars that cannot be overridden, and mounts `/agyn` + `/workspace` via an extra init container to ensure the directories exist and are writable for non-root images. The spec now says the orchestrator must not inject either variable and must not reserve either name — the image default or a user-defined [ENV](../architecture/resource-definitions.md#env) applies, and agent-specific fallbacks live in [`agynd`](../architecture/agynd-cli.md).

`agynd` does not currently fall back to `HOME=/tmp` for Codex when `HOME` is empty in the subprocess environment, and its init-script runner defaults the working directory to `/workspace` instead of "`WORKSPACE_DIR` if defined, otherwise `/tmp`".

## Acceptance Signal

- `agynio/agents-orchestrator` no longer injects `HOME` or `WORKSPACE_DIR`, no longer lists them in the reserved env name list, and no longer adds the `/agyn` + `/workspace` mounts or the permission-fixing init container introduced in [agynio/agents-orchestrator#156](https://github.com/agynio/agents-orchestrator/pull/156).
- `agynio/agynd-cli` sets `HOME=/tmp` for the Codex subprocess when `HOME` is empty in its environment; leaves `HOME` as-is for other SDKs.
- `agynio/agynd-cli` runs each init script with its working directory set to `WORKSPACE_DIR` when that variable is defined in the subprocess environment, and to `/tmp` otherwise.

## Notes

Replaces the approach merged as [agynio/agents-orchestrator#156](https://github.com/agynio/agents-orchestrator/pull/156) (closing [agynio/agents-orchestrator#155](https://github.com/agynio/agents-orchestrator/issues/155)). The earlier sequence — [#56](https://github.com/agynio/agents-orchestrator/pull/56) introducing `HOME`/`WORKSPACE_DIR` injection, [#126](https://github.com/agynio/agents-orchestrator/pull/126) changing `HOME` to `/tmp`, [#147](https://github.com/agynio/agents-orchestrator/pull/147) making both reserved — is reverted in favor of leaving `HOME` to the image or user and keeping agent-specific init-time behavior in `agynd`.
