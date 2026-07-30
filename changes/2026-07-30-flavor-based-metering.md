# Flavor-Based Compute Metering

Compute is billed per flavor rather than per CPU-core-second and gigabyte-second. This is the phase [Flavors and Environments](../product/environments/environments.md#metering) deferred when flavors first shipped.

## Target

- [Metering — Record, Labels](../architecture/metering.md)
- [Agents Orchestrator — Metering Sampling](../architecture/agents-orchestrator.md)
- [Runners — Workload Resource](../architecture/runners.md#workload-resource)
- [Flavors and Environments — Metering](../product/environments/environments.md#metering)
- [Sandboxes](../product/sandboxes/sandboxes.md)

## Why

A flavor is the unit a workload is actually allocated. CPU and memory are two numbers inside it, so billing them separately re-derives a shape the platform already names — in units nobody prices. Cloud billing prices instance types, not cores and gigabytes summed independently, and a catalog entry is the platform's instance type.

The current records are also not measurements. `allocated_cpu_millicores` and `allocated_ram_bytes` are the flavor's own numbers plus the inline resources of MCP and hook sidecars; nothing samples real consumption. So the existing unit conveys no information the flavor does not, while implying a precision it never had.

## Delta

### Metering Service

- No `FLAVOR_SECONDS` unit.
- **No `flavor` or `runner_id` column.** Labels are not stored generically: `UsageEvent` denormalizes each queryable label into its own column, so a key the service does not recognise is accepted on the wire and then dropped. Emitting `flavor` as a label without adding a column would produce records that bill nothing and cannot be grouped — the failure would be silent, and only visible as usage quietly going missing.
- Needs both columns plus `(org_id, unit, flavor, runner_id, timestamp)`, which is the aggregation billing actually reads.

`CORE_SECONDS` and `GB_SECONDS` stay in the contract and the service keeps storing and serving them. Only workload compute stops emitting them; storage still does.

### Runners Service

- The workload record has no `flavor`. The Orchestrator resolves one at start time and discards it, so nothing can attribute a workload to the tier it occupied — and a workload cannot keep billing what it started on once the environment is repointed.
- Needs a nullable `flavor` column, set at `CreateWorkload` and never rewritten.

### Agents Orchestrator

- Emits `CORE_SECONDS` and `GB_SECONDS` per workload per interval, valued from `allocated_cpu_millicores` / `allocated_ram_bytes`.
- Should emit one `FLAVOR_SECONDS` record per workload per interval, valued as the interval duration, labelled `flavor` and `runner_id`, read from the workload record rather than re-resolved.
- A workload with no flavor emits no compute record at all. That is every agent still carrying an inline image and resources; they are deprecated, and the decision is to stop metering them rather than keep a second billing shape alive for them.
- Storage sampling is unchanged.

### API

- `Unit` has no `FLAVOR_SECONDS` value. Additive.
- `Workload` has no `flavor` field. Additive.
- `allocated_cpu_millicores` and `allocated_ram_bytes` stay on `Workload` — they are what the deprecated path still reads, and removing them is a separate change from stopping their use in billing.

### Console

- The usage view breaks compute down by CPU and RAM. It should show flavor-time per tier.

## Open

- **Sidecars.** MCP and hook sidecars currently add their inline resources on top of the agent's, and that sum is what gets billed. Under flavor billing they are inside the flavor the workload occupies, so they stop being separately billable. Whether a sidecar-heavy workload should occupy a larger flavor instead is a scheduling question this does not answer.
- **Storage.** Volumes still bill raw `GB_SECONDS`. Moving them to `storage_class`/`runner_id` is the same argument applied to the storage catalog, and is not in this change.
- **History.** Existing `CORE_SECONDS`/`GB_SECONDS` rows are left as they are. Any pricing that spans the cutover reads two shapes.
