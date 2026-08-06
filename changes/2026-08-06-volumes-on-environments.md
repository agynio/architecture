# Volumes, MCPs, and Init Scripts on Environments

## Target

- [Resource Definitions — Volume](../architecture/resource-definitions.md#volume)
- [Resource Definitions — Environment](../architecture/resource-definitions.md#environment)
- [Resource Definitions — MCP](../architecture/resource-definitions.md#mcp)
- [Resource Definitions — InitScript](../architecture/resource-definitions.md#initscript)
- [Flavors and Environments — What an Environment Contains](../product/environments/environments.md#what-an-environment-contains)
- [Flavors and Environments — Who Can Use an Environment](../product/environments/environments.md#who-can-use-an-environment)
- [Sandboxes — Storage](../product/sandboxes/sandboxes.md#storage)
- [Authorization — environment](../architecture/authz.md#environment)
- [Agents Service — Environment Roles](../architecture/agents-service.md#environment-roles)
- [Agents Orchestrator — Workload Spec Assembly](../architecture/agents-orchestrator.md#workload-spec-assembly)
- [Runners — Volume Resource](../architecture/runners.md#volume-resource)
- [Agent State](../architecture/agent/state.md)

## Delta

### Volumes

Volumes are org-scoped resources holding a mount path, a size, and a persistence flag, mounted into containers through a separate `VolumeAttachment` record targeting an agent or an MCP. The entity describes nothing that exists until something mounts it, has no `name` field despite being listed by name in the Console, Terraform, and `ListVolumes`, and lets storage be declared for a workload it has no relationship to.

Both `Volume` and `VolumeAttachment` in their current form must go. The target is a `Volume` sub-resource belonging to exactly one target — an `Environment` or an `MCP` — carrying `name`, `mount_path`, `persistent`, `size`, `storage_class`, and `ttl`, with no attachment record anywhere. Agents declare no volumes at all.

- A volume is a definition, not a disk: one disk is provisioned **per owner** — per agent instance, per sandbox. Nothing is shared between owners.
- `name` is new, unique within its target, and deliberately reusable across environments.
- An MCP mounts environment volumes by listing names in a new `shared_volumes` field, resolved at workload assembly against the environment the workload is running in, mounted at the paths the main container uses. This is the only cross-container sharing the model has, and the only thing the old `Volume` entity was actually needed for.
- Mount-path collisions between an MCP's own volumes and its shared ones fail scheduling; an unresolvable shared name does the same.

### No default persistent storage

The [Sandbox](../architecture/resource-definitions.md#sandbox) currently gets an implicit `10Gi` PVC at `/workspace`, provisioned by the Orchestrator with `volume_id` NULL and no definition behind it. That must go: a sandbox mounts what its environment declares and nothing else, and `volume_id` on the [Runners volume record](../architecture/runners.md#volume-resource) becomes non-nullable — the platform provisions no storage of its own.

An environment declaring no persistent volume therefore yields workloads that keep nothing across a stop, for agents and sandboxes alike. This is the intended behavior, not a gap: [agent state](../architecture/agent/state.md) persists only where an environment provides for it. Because the consequence is invisible at a shell, `agyn sandbox start`/`stop` and the Console must say so when the environment declares no persistent volume.

### MCPs and init scripts on environments

`MCP` targets only an agent; `InitScript` targets only an agent or an MCP. Both must accept an environment as a target:

- Environment-level MCPs and init scripts apply to **every** workload running the environment, sandboxes included — which is what makes a sandbox a faithful copy of an agent's runtime rather than an approximation.
- Agent-level MCPs merge with the environment's by name, agent winning. Init scripts never collide: the environment's run first, then the agent's, each in creation order.
- `agynd` fetches both scopes at startup; a sandbox fetches the environment scope only.

### Environment permissions

Environments have **no OpenFGA type at all** today, so everything about them is gated by org ownership. That must become the same two-layer model agents use, plus one relation agents do not need:

- A new `environment` type with `owner` / `maintainer` / `user` roles and `can_use`, `can_read_config`, `can_edit_config`, `can_manage_roles`, `can_delete`.
- A new `availability` field (`internal` / `private`) driving `internal_access`, exactly as on agents.
- A new `can_create_environment` org permission, computed from `member` — any member may author an environment and becomes its `owner`.
- `can_use` is checked on `CreateSandbox` and on any `CreateAgent`/`UpdateAgent` naming the environment. A shell in a sandbox reaches the environment's secrets, egress credentials, and volume contents, so running in one is a grant rather than a consequence of visibility.
- `SetEnvironmentRole` / `RemoveEnvironmentRole` / `ListEnvironmentRoles`, mirroring the agent role API.

### Configuration change propagation

Only `agent.updated` exists. Environment sub-resources affect every agent and sandbox running the environment, and fanning that event out per referencing agent is the wrong shape. A new `environment.updated` event on an `environment:{id}` room is required, `updated_at` must propagate transitively from environment sub-resources to the environment, and the Orchestrator's [start decision](../architecture/agents-orchestrator.md#start-decision) must compare the later of `class.updated_at` and `environment.updated_at`.

## Acceptance Signal

- No `volumes` table of org-scoped definitions and no `volume_attachments` table exist. Volumes carry `environment_id` XOR `mcp_id`, and a `name`.
- An environment with a `workspace` volume at `/workspace` and an MCP listing `shared_volumes: ["workspace"]` produces a pod where the main container and that sidecar mount the same PVC at `/workspace`, and the agent's writes are visible to the MCP process.
- Two agent instances in one environment, and a sandbox started against it, each hold a distinct PVC for the same definition. No PVC is mounted by two workloads.
- A sandbox in an environment with no persistent volume starts, accepts a shell, and comes back empty after an idle stop — with the CLI having said so at `start` and warned at `stop`.
- `Runners.ListVolumes` returns no record with a NULL `volume_id`.
- An org member who is not an org owner creates an environment, grants another identity `user`, and that identity starts a sandbox in it; a third identity with no role is refused on `CreateSandbox` against a `private` environment.
- Adding a volume to an environment retries the failed instances of every agent running it on the next tick, from a single `environment.updated` event.
- A sandbox and an agent workload in the same environment run the same MCP sidecars.

## Notes

- **No migration.** Existing volume definitions and attachments are dropped rather than translated; operators redeclare storage on the environments that need it. Carrying a translation of a model that described nothing costs more than redoing it.
- Sharing storage *across* workloads remains out of the model. RWX-mounted volumes and the operator's own storage reached through [Private Networks](../architecture/private-networks.md) are the answers, as they already are for [sandbox sync](../architecture/sandbox-sync.md).
- Per-sidecar mount-path overrides for shared volumes were considered and rejected: sharing means the same files at the same path, and a divergent path is a debugging trap.
- This supersedes the sandbox workspace volume described in [Sandboxes](2026-07-15-sandboxes.md) and the runtime-only volume ownership added in [Runner-reported catalogs](2026-07-18-runner-reported-catalogs.md).
