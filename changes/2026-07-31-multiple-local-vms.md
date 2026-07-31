# More Than One Local VM

## Target

- [agyn-cli — Local Platform Commands](../architecture/agyn-cli.md#local-platform-commands-agyn-local)
- [Local Bundle — Runtime](../architecture/operations/local-bundle.md#runtime)

## Delta

There is exactly one local VM per machine, named `agyn`, and no way to address another.

- Everything derived from the VM is a constant rather than a name: the Lima instance, the `local` profile, the `agyn-local` kubeconfig context, the CA file path, and the host ports `2496`/`6445`.
- Settings live in a single flat `local:` block, which can describe only one VM.
- Two tasks are therefore impossible: moving data between two platform versions when an [in-place upgrade](2026-07-29-local-upgrade-in-place.md) cannot bridge them, and running separate clusters side by side while developing against them.

Desired state: a VM is named, and everything derived from it is named with it. `--instance NAME` addresses one for a single command; `agyn local select` chooses interactively and remembers; `agyn local use NAME` does the same without a prompt; `agyn local list` shows what exists. Resolution is `--instance`, then the stored selection, then the default.

The single-VM case is unchanged in every visible respect: no flag, no selection, and the default VM keeps the plain `agyn` / `local` / `agyn-local` names it already had. A pre-existing flat `local:` config is migrated on load.

Further VMs are given free host ports rather than the well-known pair, skipping ports already listening or claimed by another configured VM.

## Acceptance Signal

A machine with one VM behaves exactly as before: no flag names it, its profile is `local`, its context is `agyn-local`, and its config file keeps working without being edited.

`agyn local start --instance v2` creates a second VM without disturbing the first, on ports neither the first VM nor anything else on the machine is using.

`agyn local select` lists the VMs and stores the choice; subsequent commands act on it with no flag.

Each VM's `agyn` commands reach that VM's Gateway, with that VM's token and CA.

Deleting one VM leaves the other's profile, context, CA and data intact.

## Notes

- Interactive choices — VM, profile, organization — are made by moving a highlight with the arrow keys rather than typing a number from a list. Typing a number requires finding it, matching it to a row and typing it correctly, and commits the moment it is typed.
- `agyn profile select` is added alongside; profiles previously had `use NAME` but no interactive picker, which is the case where the names are least likely to be remembered.
- Downloaded images stay shared between VMs on the same version: Lima copies the disk into each instance, so nothing is gained by downloading twice.
