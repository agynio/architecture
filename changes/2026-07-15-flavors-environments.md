# Flavors and Environments

## Target

- [Flavors and Environments](../product/environments/environments.md)
- [Resource Definitions — Flavor, Environment, Agent, ENV, ImagePullSecretAttachment, EgressRuleAttachment](../architecture/resource-definitions.md)
- [Runners — Flavors, Runner Selection](../architecture/runners.md#flavors)
- [Agents Orchestrator — Runner Selection, Workload Spec Assembly, Metering](../architecture/agents-orchestrator.md)
- [Egress Gateway (product) — Rule Attachment](../product/egress-gateway/egress-gateway.md)

## Delta

### Runners Service

- The Flavor resource does not exist: no storage, no `CreateFlavor`/`UpdateFlavor`/`ListFlavors` RPCs, no delete-vs-deprecate enforcement, no per-runner uniqueness or single-`default` constraint.
- Runner selection is label-based; flavor-determined placement (environment → flavor → runner, validate-only) does not exist. Runner `labels` still participate in placement.

### Agents Service

- The Environment resource does not exist: no CRUD, no flavor/runner visibility validation on create/update, no deletion guard while agents or sandboxes reference it.
- Agents carry inline `image`, `resources`, and `runner_labels` instead of `environment_id`. No validation that the environment's flavor runner advertises the agent's `capabilities`.
- ENV and ImagePullSecretAttachment accept only `agent_id`/`mcp_id`/`hook_id` targets — no `environment_id` target.
- No migration exists from inline `(image, resources, runner_labels)` to generated per-org environments mapped to flavors.

### EgressRules Service

- EgressRuleAttachment supports only `agent_id` targets. `environment_id` targets — including the OpenZiti Dial policy shape that grants every workload identity running the environment — do not exist.

### Agents Orchestrator

- Workload spec assembly reads image and compute resources from the agent definition, not from the environment/flavor. Environment-targeted ENVs, image pull secrets, and egress attachments are not collected.
- Workload identities are not stamped with the `environment-<id>` role attribute that environment-targeted egress Dial policies match.
- Metering sources `allocated_cpu`/`allocated_ram_gb` from inline agent resources instead of the flavor.

### CLIs

- `agyn` has no flavor management commands (`agyn runners flavors list/create/update/deprecate`) and no environment management commands (`agyn environments list/create/update/delete`).

### Authz

- No OpenFGA types/relations for flavors (manage follows runner ownership) or environments (org-owner managed, member listable).

### Console

- No environment or flavor management surfaces, no unschedulable-environment flagging after runner removal.

### Terraform Provider

- No `flavor` or `environment` resources; the `agent` resource still exposes inline `image`/`resources`/`runner_labels` instead of `environment_id`.

## Acceptance Signal

- A cluster admin defines flavors on a shared runner; an org owner registers their own runner and defines a custom flavor on it; an environment referencing a flavor on another org's runner is rejected at create time.
- An agent is created with only `environment_id` (no inline image/resources/runner_labels) and its workload starts on the flavor's runner with the flavor's resources; metering records reflect the flavor allocation.
- A secret-backed ENV and an egress rule attached to an environment are observable in workloads of every agent using that environment (env var present; matched destination intercepted/injected).
- A flavor referenced by an environment cannot be deleted, only deprecated; removing a runner that backs environments flags those environments as unschedulable instead of rerouting.

## Notes

- Flavor-denominated metering (`FLAVOR_SECONDS`, flavor/runner labels) is deliberately deferred to a next phase; this change keeps `CORE_SECONDS`/`GB_SECONDS` sourced from the flavor allocation. See [Flavors and Environments — Metering](../product/environments/environments.md#metering).
- MCP and Hook sidecars keep inline `image`/`resources`; adopting flavors for sidecars is out of scope.
- [Sandboxes](2026-07-15-sandboxes.md) depend on this change — a sandbox specifies an environment.
