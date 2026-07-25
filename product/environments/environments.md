# Flavors and Environments

## Purpose

Flavors and environments separate "where and how big a workload runs" from "what the workload does."

- **Flavor** — a named compute size (CPU/memory) offered by a specific runner, declared in the runner's own configuration and reported by the runner to the platform. Flavors replace free-form per-agent CPU/memory configuration with a curated menu, the way cloud providers offer instance types.
- **Storage class** — a named storage tier offered by a specific runner, declared and reported the same way. Persistent [volumes](../../architecture/resource-definitions.md#volume) pick a class the way workloads pick a flavor.
- **Environment** — an organization-level runtime definition: a runner, a flavor on that runner, and a container image. Environments are the unit both [agents](../../architecture/resource-definitions.md#agent) and [sandboxes](../sandboxes/sandboxes.md) run in, and a shared attachment target for egress rules, environment variables, and image pull secrets.

Before this feature, every agent carried its own inline `image`, `resources`, and `runner_labels`. That put capacity decisions in the hands of every agent author, made sizes inconsistent across an organization, and left nothing for a non-agent workload (a sandbox) to run as.

## User Stories

- As a runner operator, I want to declare the compute sizes and storage tiers my runner offers in the runner's own deployment config, so the menu lives next to the implementation that honors it and is managed with the same GitOps tooling.
- As an organization owner running my own runner, I want to offer custom flavors on it — including sizes the shared runners don't offer.
- As an organization owner, I want to define an environment (image + size) once and have many agents — and engineers' sandboxes — run in it.
- As a platform admin, I want to apply platform resources and runner configuration in either order — an environment may name a flavor before the runner first reports it.
- As an operator, I want to attach egress rules and secret-backed environment variables to an environment, so everything running in it (agent or sandbox) gets the same network policy and credentials.
- As a platform operator, I want usage metered against the flavor a workload held, so billing can be priced per size tier.

## Concepts

| Term | Definition |
|---|---|
| **Runner catalog** | The set of flavors, storage classes, and capabilities a runner offers. Declared in the runner's deployment configuration and reported by the runner; not managed through platform APIs. |
| **Flavor** | A named compute size (requests/limits for CPU and memory) belonging to one runner's catalog. |
| **Storage class** | A named storage tier belonging to one runner's catalog. What backs it is the runner's concern. |
| **Environment** | An organization-scoped runtime definition: a runner + a flavor name + one container image. Referenced by agents and sandboxes. Egress rules, ENV variables, and image pull secrets can be attached to it. |

## The Runner Catalog

A flavor or storage class belongs to exactly one runner — deliberately. Runners exist to offer different capabilities: CPU architectures, devices, memory or storage types and amounts. A compute size or storage tier is only meaningful relative to the specific runner's infrastructure, so there is no cluster-wide catalog; "the same size on two runners" is two flavors.

The catalog is **declared in the runner's own deployment configuration and reported by the runner** (at startup and on config change) — not created through platform APIs. The runner is in charge of implementing every entry it offers, so the declaration lives next to the implementation: the k8s-runner's config maps each storage class to a Kubernetes StorageClass, and its flavors to pod resource requests/limits. Whoever operates the runner deployment — cluster operators for shared runners, organization teams for their own — controls the catalog through that config; the platform only reflects it:

| Runner scope | Where its catalog is managed | Who can use it |
|---|---|---|
| Cluster-scoped | The runner's deployment config (cluster operators) | Environments and volumes in any organization |
| Org-scoped | The runner's deployment config (the owning organization) | Environments and volumes in that organization only |

### Flavors

- Flavor names are unique per runner (e.g., `small`, `standard`, `large`).
- A flavor defines Kubernetes-style compute resources: `requests_cpu`, `requests_memory`, `limits_cpu`, `limits_memory`.
- At most one flavor per runner is marked `default` — used when an environment names no flavor.
- A flavor can be marked **deprecated** — a soft signal: pickers warn against new references, existing references keep scheduling.
- Removing a flavor from the runner's config removes it from the catalog; environments still naming it become unschedulable and are flagged.

### Storage Classes

- Storage class names are unique per runner (e.g., `standard`, `fast-ssd`); `default` and `deprecated` work exactly as for flavors.
- A persistent [volume](../../architecture/resource-definitions.md#volume) may name a storage class; the name is resolved on the runner the workload lands on when the volume is first provisioned. A volume that names none gets the runner's default class — as does every sandbox workspace volume.
- The class applies at provisioning time only. Already-provisioned volumes keep their storage, whatever happens to the catalog.

## Environments

An environment is an org-scoped resource managed by organization owners:

| Field | Meaning |
|---|---|
| `name` | Unique within the organization |
| `runner` | The runner workloads run on. Must be visible to the organization — validated at create/update |
| `flavor` | Name of a flavor in that runner's catalog. Not validated at create/update — resolved at every workload start. Empty means the runner's default flavor |
| `image` | Container image for the main container |

The platform init image is not part of the environment — it remains a platform concern.

### Attachments

Environments are an attachment target alongside agents:

- **Egress rules** — a rule attachment targets an agent *or* an environment. Environment-attached rules apply to every workload running the environment — agent workloads and sandboxes alike. Effective rules for a workload are the union of its agent's attachments (if it has an agent) and its environment's attachments. This is how sandboxes get network policy: they have no agent identity to attach rules to.
- **ENV variables** — an ENV resource (plain value or secret-backed) can target an environment. It is injected into the main container of every workload running that environment. On name collision, an agent-level ENV overrides the environment-level one.
- **Image pull secrets** — an image pull secret attachment can target an environment, providing registry credentials for the environment's image.

Anything attached to an environment is, in effect, available to everyone who can run a workload in it — including engineers starting [sandboxes](../sandboxes/sandboxes.md). Environments are the intentional sharing boundary. Credentials that only a specific agent should hold belong in agent-level attachments, not environment-level ones.

## Placement

An environment references its runner directly — **the environment fully determines placement**. The flavor name selects a size within that runner's catalog, late-bound at workload start:

1. **Environment create/update** — the runner must be visible to the organization (cluster-scoped, or org-scoped to the same org); a wrong runner pairing is rejected here. The flavor name is deliberately **not** validated — an environment may name a flavor the runner hasn't reported yet, so platform resources and runner config can be applied in either order. Console and CLI warn (not reject) when the name doesn't match anything currently reported — the guard against typos.
2. **Workload start** — the [Agents Orchestrator](../../architecture/agents-orchestrator.md) schedules the workload only on the environment's runner, resolving the flavor name (and every volume's storage class name) against the runner's reported catalog. There is no fallback to another runner — a different runner has no contract to honor the names. If the runner is unavailable or a name doesn't resolve, the standard start retry policy applies and the failure is surfaced on the workload.
3. **Unschedulable flagging** — removing a runner that still backs environments, or removing a flavor from a runner's config while environments still name it, leaves those environments (and the agents and sandboxes referencing them) unschedulable. They are flagged in the Console and CLI rather than silently rescheduled.

Organizations that ran the same agent on different hardware tiers (e.g., staging vs. production runners) express that as two environments.

## Agent Changes

- Agents reference an environment (`environment_id`) instead of carrying inline `image`, `resources`, and `runner_labels`. The orchestrator resolves environment → image + runner + flavor at workload spec assembly.
- `runner_labels` are removed from the agent: placement intent now lives entirely in the environment. Agent `capabilities` remain — the environment's runner must advertise (report) every capability the agent requires, checked at workload start; Console and CLI warn at agent create/update when the runner's current report doesn't cover them.
- Agent-level egress rule attachments, ENVs, and image pull secret attachments continue to work and compose with the environment's (union; agent-level ENV wins on name conflict).
- MCP and Hook sidecars keep their own inline `image`/`resources` — adopting flavors for sidecars is out of scope.

### Migration

Flavor sets are defined per runner before migration. Each existing agent is migrated by generating one environment per distinct `(image, resources)` pair in its organization, mapped to the smallest flavor on the agent's current runner that covers its requests. Inline fields are removed afterward.

## Metering

Metering keeps its current metrics for now: workloads emit `CORE_SECONDS` and `GB_SECONDS`, with allocations sourced from the flavor's resource definition instead of inline agent resources.

**Next phase:** metering moves to flavor-denominated usage — records dimensioned by `flavor_name`/`runner_id` with a `FLAVOR_SECONDS` unit (and storage by `storage_class`/`runner_id`), so usage can be priced per tier the way cloud billing is priced per instance type.

## Lifecycle

| Event | Effect |
|---|---|
| Runner reports its catalog | Reported flavors and storage classes become resolvable immediately (per runner visibility). Environments that already named them start scheduling. |
| Flavor/class added to runner config | In the catalog on the next report. |
| Flavor/class changed in runner config | New values apply to workloads started (volumes provisioned) after the next report. Running workloads are not resized; existing volumes keep their storage. |
| Flavor/class marked deprecated | Soft signal: pickers warn against new references. Existing references keep scheduling. |
| Flavor/class removed from runner config | Gone from the catalog on the next report. Environments/volumes still naming it become unschedulable and are flagged. Renames are a removal plus an addition. |
| Environment created | Available to agents and sandboxes in the organization — even if its flavor name isn't reported yet (it won't schedule until it is). |
| Environment updated (image, runner, or flavor) | Applies to workloads started after the change. Running workloads are not restarted. |
| Environment deleted | Fails while any agent or sandbox references it. |
| Runner removed | Environments referencing it become unschedulable and are flagged. |

## Constraints

- A flavor or storage class belongs to exactly one runner's catalog; there is no cross-runner identity. Two runners offering "the same" size are two flavors, and references are meaningful only relative to the referenced runner.
- Catalog entries are referenced by name and resolved at workload start / volume provisioning — never by stored ID, never validated at resource create time.
- An environment references exactly one runner, at most one flavor name, and one image. Multi-image environments and per-environment init images are out of scope.
- Running workloads never pick up catalog or environment changes in place — changes apply on next workload start. Provisioned volumes never change storage class.
- MCP and Hook resources are unaffected by this feature.

## Related Architecture

- [Resource Definitions — Flavor](../../architecture/resource-definitions.md#flavor)
- [Resource Definitions — Storage Class](../../architecture/resource-definitions.md#storage-class)
- [Resource Definitions — Environment](../../architecture/resource-definitions.md#environment)
- [Runners — Runner Catalog](../../architecture/runners.md#runner-catalog)
- [Runners — Runner Selection](../../architecture/runners.md#runner-selection)
- [Agents Orchestrator](../../architecture/agents-orchestrator.md)
- [Egress Gateway](../egress-gateway/egress-gateway.md)
- [Secrets](../../architecture/secrets.md)
