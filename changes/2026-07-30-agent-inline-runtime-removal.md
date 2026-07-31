# Removing Inline Image and Compute from the Agent

**Date:** 2026-07-30

## Target

- [Resource Definitions — Agent](../architecture/resource-definitions.md#agent)
- [Agents Service](../architecture/agents-service.md)
- [Agents Orchestrator — Workload Spec Assembly](../architecture/agents-orchestrator.md)
- [Flavors and Environments — Agent Changes](../product/environments/environments.md#agent-changes)
- [Flavor-Based Compute Metering](2026-07-30-flavor-based-metering.md)

## Why

The target shape is already written: [Resource Definitions](../architecture/resource-definitions.md#agent) lists `environment_id` with no `image` or `resources` row. [Flavors and Environments](2026-07-15-flavors-environments.md) kept the inline fields only for migration — "Inline fields are removed afterward". This is that step.

Keeping both is not neutral. An agent with no `environment_id` resolves to no flavor and no runner, drops to label-based placement, and under [flavor-based metering](2026-07-30-flavor-based-metering.md) emits no compute record. The deprecated path is the one that silently bills nothing, and it stays reachable while `image` is writable.

`resources` sizes nothing: `ContainerSpec` has no resources field, so the agent's requests never reach a container. Their only runtime consumer is the allocation figure flavor-based metering retires.

## Delta

### API

- `Agent`, `CreateAgentRequest`, and `UpdateAgentRequest` still carry `image` and `resources` as `[deprecated = true]`. They go, and their wire numbers stay retired. Breaking, unlike the neighbouring flavor changes.
- `environment_id` becomes required on create, and update must reject the empty string rather than clearing it.
- `ComputeResources` stays — MCP and Hook keep inline image and resources.

### Agents Service

- `CreateAgent` requires `init_image` but not `environment_id`, and explicitly allows a missing one so the agent "may run from the deprecated inline image and resources instead".
- `AgentInput` and `agentColumns` carry `image` plus four `resources_*` columns. All six go, and `agents.environment_id` — nullable since `0018_agent_environment.sql` — ends up `NOT NULL`.

### Agents Orchestrator

- Assembly seeds `mainImage := agent.GetImage()` and overwrites it only when an environment exists.
- `resolveAgentEnvironment` returns `(nil, nil, nil)` for an agent with no environment. Proto getters are nil-safe, so the result is a workload with empty runner and flavor — a silent fallback rather than a refusal. It becomes an error.
- `sumAllocatedResources` adds `agent.GetResources()` to the sidecar totals; the agent term drops. No workload changes size.

### Migration

Every agent with `environment_id IS NULL` needs one first. [Flavors and Environments](../product/environments/environments.md#migration) already specifies the shape — one generated environment per distinct `(image, resources)` pair per organization, mapped to the smallest covering flavor on the agent's runner. This change runs it to completion.

### Terraform Provider

A blocking dependency, not a follow-on. `agent_resource.go` has `image` as `Required`, a `resources` block, and **no `environment_id` attribute at all**; there is no environment resource. The provider can only express the deprecated shape, so Terraform-managed agents have no path off it until both land.

### CLI

`agyn sandbox` documents `--agent @handle` as not implementable "because the Agent resource still carries an inline image and has no environment_id". Already stale; implementable once every agent has one.

### Console

Agent create collects Image and a compute editor beside an **optional** Environment select, and the configuration tab edits both. Environment becomes required; the other two go.

## Acceptance Signal

- `CreateAgent` without `environment_id` fails with `InvalidArgument`; `UpdateAgent` cannot clear one.
- No agent row has a null `environment_id`, and the `image` and `resources_*` columns are gone.
- Every assembled agent workload takes its image from an environment and carries a non-empty runner and flavor.
- The Terraform agent resource takes `environment_id` and has no `image` or `resources`, with an `environment` resource to point at.
- Console agent create and edit have no image or compute inputs, and Environment cannot be left unset.
- `agyn sandbox --agent @handle` resolves the agent's environment.

## Open

- **Organizations with no runner.** Nothing to map an environment to, so their agents cannot satisfy a `NOT NULL` `environment_id`. Migrate them to an environment naming an unreported flavor (created but unschedulable, which environments already tolerate), or hold them back?
- **`init_image` stays on the agent.** `Environment` has none, and sandboxes take theirs from orchestrator config. The runtime definition stays split; unifying it is its own change.
- **`Workload.allocated_cpu_millicores` / `allocated_ram_bytes`.** Once no agent carries inline resources only sidecars contribute. Removing them stays separate, as [flavor-based metering](2026-07-30-flavor-based-metering.md) already said.
