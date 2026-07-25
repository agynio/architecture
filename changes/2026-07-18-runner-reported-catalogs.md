# Runner-Reported Catalogs and Storage Classes

Supersedes the flavor-management model of [Flavors and Environments](2026-07-15-flavors-environments.md): flavors are no longer platform-managed resources referenced by ID — they are entries in a catalog the runner declares in its own configuration and reports to the platform, referenced by name with late binding. Storage classes join the catalog as the storage counterpart of flavors.

## Target

- [Flavors and Environments](../product/environments/environments.md)
- [Concepts — Runner Catalog, Flavor, Storage Class, Environment](../product/concepts.md)
- [Resource Definitions — Flavor, Storage Class, Environment, Volume, Agent, Capabilities](../architecture/resource-definitions.md)
- [Runners — Runner Catalog, Runner Selection, Volume Resource](../architecture/runners.md#runner-catalog)
- [Runner — Workload Model](../architecture/runner.md#workload-model)
- [k8s-runner — Runner Catalog, PersistentVolumeClaims](../architecture/k8s-runner.md#runner-catalog)
- [Agents Orchestrator — Runner Selection, Workload Spec Assembly](../architecture/agents-orchestrator.md)
- [Agents Service — Environments](../architecture/agents-service.md)
- [Sandboxes — Workspace volume](../product/sandboxes/sandboxes.md)

## Delta

### Model changes vs. the 2026-07-15 spec

- **Catalog is runner-declared, not platform-managed.** `CreateFlavor`/`UpdateFlavor` RPCs, the delete-vs-deprecate enforcement, and the cluster-admin/org-owner flavor management authz split are dropped. The runner declares flavors, storage classes, and capabilities in its deployment configuration and reports them via a new `ReportRunnerCatalog` RPC (service-token authenticated, like `EnrollRunner`) — at startup after enrollment and on config change. The report declaratively replaces the stored catalog; invalid reports are rejected wholesale, keeping the previous catalog.
- **References are by name, late-bound.** The Environment resource carries `runner_id` + `flavor` (a name, optional — empty resolves to the runner's default flavor) instead of `flavor_id`. Placement becomes environment → runner directly. Names are not validated at environment/volume create time (Console/CLI warn on unmatched names); the Orchestrator resolves them against the runner's reported catalog at workload start, and unresolvable names fail scheduling with the standard retry policy plus unschedulable flagging.
- **Deprecation is a soft signal.** A `deprecated` catalog entry still resolves and schedules; pickers warn. Removing an entry from the runner config is the hard path — referencing environments/volumes become unschedulable, flagged, and recover when the name reappears.
- **Capabilities move into the report.** Runner `capabilities` are no longer set at `RegisterRunner`/`UpdateRunner` (and are removed from the Terraform `agyn_runner` schema); the runner reports them. Agent capability checks happen at workload start; agent create/update only warns.
- **Storage classes are new.** A per-runner catalog section mirroring flavors (`name`, `default`, `deprecated`). Agents-service Volume definitions gain an optional `storage_class` name, resolved on the runner the workload lands on when the volume is first provisioned; null uses the runner's default class. Sandbox workspace volumes always use the default class. The k8s-runner maps each entry to a Kubernetes `storageClassName` in its config; the resolved class is recorded on the provisioned volume record (`storage_class`) and never changes retroactively.

### Runners Service

- No catalog storage, no `ReportRunnerCatalog`/`ListFlavors`/`ListStorageClasses` RPCs, no report validation (per-kind name uniqueness, single `default`).
- Runner `capabilities` are admin-set at registration instead of runner-reported.
- Volume records carry no `storage_class`.
- Runner selection: flavor-determined placement does not exist in any form — neither ID-based nor the name-resolution shape specced now.

### Agents Service

- The Environment resource does not exist (unchanged gap from 2026-07-15), and the target shape is now `runner_id` + `flavor` name with no flavor existence validation.
- Volume definitions have no `storage_class` field.

### Agents Orchestrator

- No catalog name resolution at workload start (flavor, storage classes, capabilities against the reported catalog), no unschedulable surfacing for unresolved names, no `runner_conflict` handling keyed to the environment's runner reference.
- Workload spec assembly does not pass a storage class per persistent volume; `CreateVolume` records no class.

### Runners (k8s-runner)

- No catalog section in runner configuration, no `ReportRunnerCatalog` call at startup or on config reload.
- PVCs are created with the cluster default StorageClass only — no per-volume class from the workload spec, no name → `storageClassName` mapping.

### Console / CLI

- No catalog pickers backed by `ListFlavors`/`ListStorageClasses`, no soft warnings on unmatched names, no unschedulable-environment flagging for vanished catalog entries.

## Acceptance Signal

- A runner deployed with a catalog in its config reports it at startup; `ListFlavors`/`ListStorageClasses` reflect the config with no platform API writes involved.
- An environment created *before* its runner ever reports (naming a not-yet-existing flavor) is accepted with a warning, stays unschedulable, and starts scheduling within one report + reconciliation cycle after the runner comes up with that flavor in config.
- Removing a flavor from the runner config flags referencing environments as unschedulable; re-adding it recovers them without touching platform state.
- A persistent volume with `storage_class: fast-ssd` gets a PVC with the mapped `storageClassName`; a volume without a class and a sandbox workspace volume get the runner's default entry; the resolved class appears on the provisioned volume record.
- A workload whose volume names a class absent from the runner's catalog fails to schedule with the standard retry policy and a visible reason.
- `RegisterRunner` accepts no capabilities; after the runner's first report, capability-requiring agents schedule based on the reported list.

## Notes

- Rationale: every catalog entry needs a runner-side implementation mapping anyway (flavor → resources, class → StorageClass, capability → sidecars). Platform-managed records would duplicate that declaration and drift; reporting makes the runner config the single source of truth and lets platform resources and runner config be applied in either order.
- Flavor-denominated metering remains a next phase, now dimensioned by `flavor_name`/`runner_id` (and `storage_class`/`runner_id` for storage) — catalog entries have no stable platform IDs to price against.
- Catalog entry renames are removal + addition by design; there is no rename detection.
- [Sandboxes](2026-07-15-sandboxes.md) is unaffected except that the workspace volume now takes the runner's default storage class.
