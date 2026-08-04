# Environment and Runtime Unification

## Target

- [Agent Init Container](../architecture/agent-init.md)
- [Flavors and Environments](../product/environments/environments.md)
- [Resource Definitions — Environment, Agent, MCP](../architecture/resource-definitions.md#environment)
- [Sandboxes — What's Inside](../product/sandboxes/sandboxes.md#whats-inside)
- [Agents Orchestrator — Workload Spec Assembly](../architecture/agents-orchestrator.md)

## Delta

### Init Images

- `agynd` and the [`agyn`](../architecture/agyn-cli.md) CLI are bundled into every `agent-init-*` image, so an image a user selects determines which build of the platform's own binaries a workload runs. Each moves into its own chart-shipped init image — `agynd-cli-init` and `agyn-cli-init`, built by the repository that owns the binary — injected into every workload and out of the agent runtime images entirely.
- `AGYN_VERSION` and `AGYND_VERSION` are pinned per agent-init repository. Only the agent CLI version remains, which is what makes an image's tags correspond to something a user can choose.
- Neither `agynd-cli-init` nor `agyn-cli-init` exists. The repositories that own the binaries publish release artifacts but no images.
- The agent-init repositories are named and tagged for a shape that is going away. They become `agyn-runtime-<cli>`, pinning only their own CLI version, with tags corresponding to agent CLI versions.

### Agents Service

- `Agent.init_image` exists and is required on create. It goes; the agent runtime comes from the environment.
- `Environment.image` is a free-form string. It becomes `workspace_image_id` + `workspace_image_tag`, plus optional `agent_runtime_image_id` + `agent_runtime_image_tag`.
- `MCP.image` is a free-form string. It becomes `image_id` + `image_tag`.
- No validation that an image reference resolves to a visible image of the type the slot requires, with a tag that exists.
- `CreateAgent` does not require that the named environment has an agent runtime image, so an agent can be created with no agent CLI to run.
- Environments are not flagged unschedulable when a referenced image is deleted, its tag goes `gone`, or its visibility narrows.

### Agents Orchestrator

- Assembly reads `init_image` from the agent definition and falls back to `DEFAULT_INIT_IMAGE`. Both go: the two platform init containers come from orchestrator configuration and are unconditional, and the agent runtime init container comes from the environment when it names one.
- Only one init container is built. Three are needed, and a workspace-only environment gets just the two platform ones.

### Sandboxes

The platform init image comes from orchestrator configuration and there is no way to specify one per sandbox, so a sandbox contains the platform binaries and nothing else — no agent CLI. The environment's agent runtime image is not injected, which is what leaves a sandbox empty of the tooling its agents use.

### Console

- The agent form collects Image and Init image. Both go; Environment becomes required and only environments with an agent runtime image are offered.
- MCP forms collect a free-form image string rather than selecting from the catalog.
- Environment forms collect one free-form image rather than a workspace image and an optional agent runtime image, each with a tag.

### Terraform Provider

`agent` still exposes `init_image`; there is no `environment` resource at all, and `mcp` takes a free-form image. The provider can only express the shape being removed.

## Acceptance Signal

- An agent is created with no image field of any kind, and its workload runs the environment's workspace image with the environment's agent runtime image supplying the agent CLI.
- Three init containers appear in the pod: `agynd-cli-init`, `agyn-cli-init`, and the environment's agent runtime.
- `CreateAgent` against a workspace-only environment fails.
- A sandbox started on an environment with an agent runtime image has that agent CLI on `PATH`; one started on a workspace-only environment has `agyn` and no agent CLI.
- `agynd` reads `config.json` from the agent runtime image as before — the orchestrator sets no SDK type and no binary path.
- No `DEFAULT_INIT_IMAGE` setting remains.
- Upgrading the platform moves `agynd` and `agyn` in every workload without anyone selecting a new image.

## Notes

- Migration is manual and deliberately not automated. Environments today carry one image and agents carry an init image, so an environment shared by agents with different init images becomes several environments — a fan-out, not a field rewrite. Catalog entries must exist before the cutover; afterwards, anything still holding a free-form image string does not run.
- Environment count rising with the number of agent CLIs in use is accepted. It is what keeps a single answer to what a workload in an environment contains, which is what lets a sandbox carry the same tooling as an agent there.
- `config.json` stays in the agent runtime image and keeps its current shape. The catalog therefore does not know which agent CLI a given `agent_runtime` image provides — only its name and description say so.
- Depends on [Image Catalog](2026-08-03-image-catalog.md) for the records both environments and MCPs reference.
