# In-Place Platform Upgrades for the Local Bundle

## Target

- [Local Bundle — Upgrade Model](../architecture/operations/local-bundle.md#upgrade-model)
- [agyn-cli — Local Platform Commands](../architecture/agyn-cli.md#local-platform-commands-agyn-local)

## Delta

`agyn local upgrade` recreates the VM from a newer image. Everything in it — databases, threads, agents, workloads — is discarded, so the command is destructive and gated behind a confirmation.

- There is no way to move to a newer platform release without losing the VM's contents. The only upgrade path is image replacement.
- The image is described as never reconciling or upgrading itself, with no distinction between the platform releases installed into it at bake time and the infrastructure underneath them.
- Recreating the VM has no name of its own: it is what `upgrade` does, rather than something a user asks for.

Desired state: `upgrade` moves the `agyn-platform` and `agyn-apps` Helm releases in the running VM to the newest published charts and keeps the VM. Values are reused from the installed release rather than re-rendered, because the bake configured the cluster and `start` has since written the bootstrap token and the host's chosen port into it. Infrastructure baked into the image — k3s, Istio, cert-manager, OpenZiti, Postgres — still moves only by replacing the image, spelled `delete` then `start`.

## Acceptance Signal

`agyn local upgrade` against a VM with data leaves that data in place and reports the chart versions before and after.

Running it twice in a row is a no-op the second time.

A VM whose platform predates a chart release picks that release up without being recreated; a VM whose k3s or Istio predates a new image does not.

`upgrade` never deletes the VM. Recreating it requires `delete` and `start`.

A service running from source against the VM is reported as being reset to its chart image, because an upgrade rewrites every workload the charts own.

## Notes

- Rationale: image replacement is not what "upgrade" sounds like it does, it is not reversible, and a user picking up a platform release is not asking to lose their work. Keeping it available under `delete` + `start` costs nothing and makes the destructive path explicit.
- The two layers move at different rates: the platform charts release continuously, while the image's infrastructure changes rarely. Tying them to one command forced the slow layer's cost onto the fast layer's cadence.
- The upgrade runs against the newest published charts rather than a pinned set. A local VM is not a production cluster, and pinning is available by driving Helm directly.
