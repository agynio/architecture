# Flavors and Environments

## Purpose

Flavors and environments separate "where and how big a workload runs" from "what the workload does."

- **Flavor** — a named compute size (CPU/memory) offered by a specific runner, defined by whoever operates that runner. Flavors replace free-form per-agent CPU/memory configuration with a curated menu, the way cloud providers offer instance types.
- **Environment** — an organization-level runtime definition: a flavor plus a container image. Environments are the unit both [agents](../../architecture/resource-definitions.md#agent) and [sandboxes](../sandboxes/sandboxes.md) run in, and a shared attachment target for egress rules, environment variables, and image pull secrets.

Before this feature, every agent carried its own inline `image`, `resources`, and `runner_labels`. That put capacity decisions in the hands of every agent author, made sizes inconsistent across an organization, and left nothing for a non-agent workload (a sandbox) to run as.

## User Stories

- As a cluster admin, I want to define the compute sizes available on the shared runners, so agent authors pick from a curated menu instead of requesting arbitrary CPU/memory.
- As an organization owner running my own runner, I want to define custom flavors for it — including sizes the shared runners don't offer.
- As an organization owner, I want to define an environment (image + size) once and have many agents — and engineers' sandboxes — run in it.
- As an operator, I want to attach egress rules and secret-backed environment variables to an environment, so everything running in it (agent or sandbox) gets the same network policy and credentials.
- As a platform operator, I want usage metered against the flavor a workload held, so billing can be priced per size tier.

## Concepts

| Term | Definition |
|---|---|
| **Flavor** | A named compute size (requests/limits for CPU and memory) belonging to one runner. Managed by the runner's operator: cluster admins for cluster-scoped runners, organization owners for org-scoped runners. |
| **Environment** | An organization-scoped runtime definition: one flavor + one container image. Referenced by agents and sandboxes. Egress rules, ENV variables, and image pull secrets can be attached to it. |

## Flavors

A flavor belongs to exactly one runner — deliberately. Runners exist to offer different capabilities: CPU architectures, devices, memory or storage types and amounts. A compute size is only meaningful relative to the specific runner's hardware, so there is no cluster-wide flavor catalog; "the same size on two runners" is two flavors. Flavors are managed by the runner's operator, following the existing runner-registration authorization split:

| Runner scope | Who manages its flavors | Who can use them |
|---|---|---|
| Cluster-scoped | Cluster admins | Environments in any organization |
| Org-scoped | That organization's owners | Environments in that organization only |

- Flavor names are unique per runner (e.g., `small`, `standard`, `large`).
- A flavor defines Kubernetes-style compute resources: `requests_cpu`, `requests_memory`, `limits_cpu`, `limits_memory`.
- At most one flavor per runner is marked `default` — used when an environment is created without an explicit flavor.
- A flavor referenced by any environment can be **deprecated** (blocks new environment references, existing ones keep working) but not deleted.

## Environments

An environment is an org-scoped resource managed by organization owners:

| Field | Meaning |
|---|---|
| `name` | Unique within the organization |
| `flavor_id` | The flavor (and therefore the runner) workloads run on |
| `image` | Container image for the main container |

The platform init image is not part of the environment — it remains a platform concern.

### Attachments

Environments are an attachment target alongside agents:

- **Egress rules** — a rule attachment targets an agent *or* an environment. Environment-attached rules apply to every workload running the environment — agent workloads and sandboxes alike. Effective rules for a workload are the union of its agent's attachments (if it has an agent) and its environment's attachments. This is how sandboxes get network policy: they have no agent identity to attach rules to.
- **ENV variables** — an ENV resource (plain value or secret-backed) can target an environment. It is injected into the main container of every workload running that environment. On name collision, an agent-level ENV overrides the environment-level one.
- **Image pull secrets** — an image pull secret attachment can target an environment, providing registry credentials for the environment's image.

Anything attached to an environment is, in effect, available to everyone who can run a workload in it — including engineers starting [sandboxes](../sandboxes/sandboxes.md). Environments are the intentional sharing boundary. Credentials that only a specific agent should hold belong in agent-level attachments, not environment-level ones.

## Placement

An environment references one flavor, and a flavor belongs to one runner — so **the environment fully determines placement**: environment → flavor → runner. Enforcement:

1. **Environment create/update** — the flavor must exist, must not be deprecated, and its runner must be visible to the organization (cluster-scoped, or org-scoped to the same org). This is the only place a wrong flavor/runner pairing could be introduced, so it is rejected here.
2. **Workload start** — the [Agents Orchestrator](../../architecture/agents-orchestrator.md) schedules the workload only on the flavor's runner. There is no fallback to another runner — a different runner has no contract to honor the flavor. If the runner is unavailable, the standard start retry policy applies and the failure is surfaced on the workload.
3. **Runner removal** — removing a runner that still backs environments leaves those environments, and the agents and sandboxes referencing them, unschedulable. They are flagged in the Console and CLI rather than silently rescheduled.

Organizations that ran the same agent on different hardware tiers (e.g., staging vs. production runners) express that as two environments.

## Agent Changes

- Agents reference an environment (`environment_id`) instead of carrying inline `image`, `resources`, and `runner_labels`. The orchestrator resolves environment → image + flavor at workload spec assembly.
- `runner_labels` are removed from the agent: placement intent now lives entirely in the environment. Agent `capabilities` remain — the flavor's runner must advertise every capability the agent requires, validated at agent create/update and at workload start.
- Agent-level egress rule attachments, ENVs, and image pull secret attachments continue to work and compose with the environment's (union; agent-level ENV wins on name conflict).
- MCP and Hook sidecars keep their own inline `image`/`resources` — adopting flavors for sidecars is out of scope.

### Migration

Flavor sets are defined per runner before migration. Each existing agent is migrated by generating one environment per distinct `(image, resources)` pair in its organization, mapped to the smallest flavor on the agent's current runner that covers its requests. Inline fields are removed afterward.

## Metering

Metering keeps its current metrics for now: workloads emit `CORE_SECONDS` and `GB_SECONDS`, with allocations sourced from the flavor's resource definition instead of inline agent resources.

**Next phase:** metering moves to flavor-denominated usage — records dimensioned by `flavor_id`/`flavor_name`/`runner_id` with a `FLAVOR_SECONDS` unit, so usage can be priced per flavor the way cloud billing is priced per instance type.

## Lifecycle

| Event | Effect |
|---|---|
| Flavor created | Available to environments (per runner visibility) immediately. |
| Flavor updated | New resource values apply to workloads started after the change. Running workloads are not resized. |
| Flavor deprecated | New environment references rejected. Existing environments keep working. |
| Flavor deleted | Fails while any environment references it — deprecate instead. |
| Environment created | Available to agents and sandboxes in the organization. |
| Environment updated (image or flavor) | Applies to workloads started after the change. Running workloads are not restarted. |
| Environment deleted | Fails while any agent or sandbox references it. |
| Runner removed | Environments backed by its flavors become unschedulable and are flagged. |

## Constraints

- A flavor belongs to exactly one runner; there is no cross-runner flavor identity. Two runners offering "the same" size are two flavors.
- An environment references exactly one flavor and one image. Multi-image environments and per-environment init images are out of scope.
- Running workloads never pick up flavor or environment changes in place — changes apply on next workload start.
- MCP and Hook resources are unaffected by this feature.

## Related Architecture

- [Resource Definitions — Flavor](../../architecture/resource-definitions.md#flavor)
- [Resource Definitions — Environment](../../architecture/resource-definitions.md#environment)
- [Runners — Flavors](../../architecture/runners.md#flavors)
- [Runners — Runner Selection](../../architecture/runners.md#runner-selection)
- [Agents Orchestrator](../../architecture/agents-orchestrator.md)
- [Egress Gateway](../egress-gateway/egress-gateway.md)
- [Secrets](../../architecture/secrets.md)
