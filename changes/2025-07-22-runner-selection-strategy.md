# 2025-07-22: Runner Selection Strategy

## Decision

The [Agents Orchestrator](../architecture/agents-orchestrator.md) selects runners using **organization scoping + label matching + random selection**.

Runners carry key-value `labels` describing their capabilities (e.g., `gpu: "true"`, `region: "eu-west-1"`). Agents carry an optional `runner_labels` field declaring required runner properties.

## Selection Algorithm

1. **Scope filtering** — org-scoped runners for the agent's organization + cluster-scoped runners. Only `enrolled` runners.
2. **Label matching** — if the agent defines `runner_labels`, keep only runners whose `labels` contain all required key-value pairs (exact match).
3. **Random selection** — pick one from the remaining set.

No match → workload fails to schedule; orchestrator retries on next reconciliation pass.

## What Changed

- `runners.md`: Added `labels` field to Runner resource, `UpdateRunner` method, and Runner Selection section.
- `resource-definitions.md`: Added `runner_labels` field to Agent resource.
- `agents-orchestrator.md`: Added Runner Selection step to Agent Start Flow, updated sequence diagram and Runners dependency description.
- `open-questions.md`: Removed "Runner Selection Strategy" question.

## Rationale

Labels provide indirection between agent requirements and runner capabilities. Unlike explicit runner ID references, labels are stable across runner reprovisioning — replacing a runner only requires the new runner to carry the same labels. The agent definition does not change.

Organization scoping is the primary isolation boundary (org-scoped runners serve only their org, cluster-scoped runners serve everyone). Labels add a secondary filtering dimension for infrastructure properties (hardware, region, compliance zones) without coupling agents to specific runner instances.

Random selection among matching runners is sufficient at current scale (few runners per scope). Capacity-aware selection can be layered on later if runner resource contention becomes a real problem.
