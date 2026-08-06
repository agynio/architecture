# agyn environments Command Group

## Target

- [agyn-cli — Environment Commands](../architecture/agyn-cli.md#environment-commands)
- [Flavors and Environments — Managing Environments](../product/environments/environments.md#managing-environments)

## Delta

`agyn` has no environment commands at all. Environments are reachable only through the Console and the [Terraform provider](../architecture/operations/terraform-provider.md), which was defensible while they were org-owner-only infrastructure and is not once any member can author one and grant roles on it — see [Volumes, MCPs, and Init Scripts on Environments](2026-08-06-volumes-on-environments.md).

The group must cover the environment record, each kind of content it owns, and its roles:

- `list` / `show` / `create` / `update` / `delete` on the environment. `create` grants the caller `owner`. An unreported flavor name warns and proceeds rather than failing, matching the late binding the placement model depends on.
- `volumes`, `mcps`, `init-scripts`, and `vars` subcommands for the environment's contents. `mcps add --share VOLUME` sets `shared_volumes`.
- `roles list` / `grant` / `revoke`.

Two behaviors are load-bearing rather than cosmetic:

- **`--size` is what makes a volume persistent.** The resource makes `size` and `persistent` biconditional (required when persistent, rejected otherwise), so a separate `--persistent` flag could only ever contradict it.
- **A caller without `can_read_config` gets a stated refusal, not an empty list.** `show` on an environment whose configuration is not visible must say so; printing nothing under "Volumes" reads as "declares no storage," which is a materially different fact and one that now determines whether work survives a restart.

`agyn environments volumes remove` must name how many provisioned disks the removal will deprovision, and `roles grant --role user` must state that the role opens an interactive shell onto the environment's secrets, egress credentials, and volume contents.

## Acceptance Signal

- A non-owner organization member runs `agyn environments create`, then `agyn environments volumes add`, and a sandbox started against that environment mounts the volume.
- `agyn environments show` on an environment the caller holds no role on prints the metadata header and an explicit "configuration not visible" line — not an empty contents listing.
- `agyn environments create --flavor` with a name the runner has not reported succeeds with a warning; `agyn environments list` marks the environment unschedulable until the runner reports it.
- `agyn environments volumes add ENV cache --path /cache` (no `--size`) produces an ephemeral volume; adding `--size 5Gi` produces a persistent one.
- `agyn environments volumes remove` on a volume with provisioned disks states the count before proceeding.
- `agyn environments roles grant ENV @user --role user` lets that identity start a sandbox in the environment and refuses beforehand.

## Notes

- `vars`, not `env`: `agyn environments env` reads as a tautology, and `--env` on `agyn sandbox start` already names the environment.
- The group is deliberately imperative rather than a second declarative path. Terraform states what an environment should be; the CLI reports and adjusts what it currently is.
- Agent-level MCPs, init scripts, and ENVs have no CLI surface either. That gap is older and untouched here.
