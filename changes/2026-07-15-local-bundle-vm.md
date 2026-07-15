# Local Bundle VM and `agyn local`

## Target

- [Local Bundle](../architecture/operations/local-bundle.md)
- [agyn-cli — Local Platform Commands](../architecture/agyn-cli.md#local-platform-commands-agyn-local)

## Delta

The two image repositories (`agynio/bundle-vm-base`, `agynio/bundle-vm`) exist with a working builder harness (Packer inheritance, install/prepull/cleanup, Lima templates), and the unified `agyn-platform` umbrella chart has been validated live on the base VM. The remaining gaps against the desired state:

- The platform image does not yet bake the complete platform: Ziti-dependent services (agents-orchestrator, runners, egress) are disabled pending OpenZiti provisioning inside the VM, and OIDC is not wired (UIs render, login does not work).
- The build process still uses Argo CD: the base image installs it and the platform build applies Argo CD Applications. The desired state has no Argo CD at any stage — the base image ships without it and the bake installs the umbrella chart directly.
- Artifacts are not published to the CDN (`downloads.agyn.cloud/bundle-vm/<version>/<arch>/`), and `latest.json` is not part of the publish flow.
- The `agyn local` command group does not exist in `agyn-cli`.

## Acceptance Signal

On a clean machine with `limactl`, `xz`, and `qemu` installed, `agyn local start` downloads the latest published image, boots the VM, installs the CA, and `https://console.agyn.dev:<port>` serves a working platform with login. `agyn local status`, `delete --purge`, `upgrade`, `doctor`, `config`, and `ca` behave as specified.

## Notes

- The per-install CA direction (Phase 2) is tracked as an [open question](../open-questions.md#local-bundle-per-install-ca); the baked-CA rotation caveat is being addressed separately.
