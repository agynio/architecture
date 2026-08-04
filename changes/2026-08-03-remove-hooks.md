# Remove Hooks

## Target

- [Resource Definitions](../architecture/resource-definitions.md)
- [Agents Service](../architecture/agents-service.md)
- [Agents Orchestrator — Workload Spec Assembly](../architecture/agents-orchestrator.md)
- [Console — Agents](../product/console/console.md#agents)

## Delta

The `Hook` resource is gone from every specification. It remains in the implementation.

### Agents Service

- The `Hook` resource and its CRUD surface still exist, along with `ListHooks` and its `Get` counterpart in the authorization table.
- `ENV`, `InitScript`, and `VolumeAttachment` still accept a `hook_id` target, and their mutual-exclusion rules still count it as a fourth (or third) option.
- `agent.updated` still fires on hook sub-resource changes.

### Agents Orchestrator

Assembly still fetches hooks, builds hook sidecar containers, injects hook ENVs, mounts volumes into them, and includes them in the containers receiving the egress CA bundle and its trust environment variables.

### Console

The agent detail page still has a Hooks tab with per-hook Edit, Environment Variables, Init Scripts and Delete actions. Volume listings still offer `hook` as an attachment kind and a filter value.

### Runners Service

Provisioned volume listings still report `hook` as an attachment kind and accept it in `filter.attached_to_kind_in`.

### Terraform Provider

`agyn_hook` still exists, and sub-resource ownership still allows `hook_id`.

### Platform Documentation

`administer/hooks.md` documents a resource that no longer exists.

## Acceptance Signal

- No API, database table, Console surface, Terraform resource, or documentation page refers to hooks.
- `ENV`, `InitScript`, and `VolumeAttachment` accept only agent and MCP targets.
- An assembled workload contains the agent container, its MCP sidecars, and platform sidecars — nothing else.

## Notes

- Hooks are removed rather than migrated. Nothing is in use, and no replacement is offered.
- Independent of the image work, but bundled with it: hooks are the last resource carrying a free-form image reference, so until they go, "no resource outside the catalog holds a registry URL" is not literally true. It is also what allows `ImagePullSecretAttachment` to be removed outright rather than kept alive for one target — see [Image Pull Proxy](2026-08-03-image-pull-proxy.md).
