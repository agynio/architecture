# Private Registry Support

## Target

- [Providers, Models, and Secrets — Image Pull Secret](../architecture/providers.md#image-pull-secret)
- [Providers, Models, and Secrets — Secret](../architecture/providers.md#secret)
- [Secrets Service — Image Pull Secret Resolution](../architecture/secrets.md#image-pull-secret-resolution)
- [Resource Definitions — Image Pull Secret Attachment](../architecture/resource-definitions.md#image-pull-secret-attachment)
- [Agents Service](../architecture/agents-service.md)
- [Agents Orchestrator — Agent Start Flow](../architecture/agents-orchestrator.md#agent-start-flow)
- [Runner — Workload Model](../architecture/runner.md#workload-model)
- [k8s-runner — Image Pull Credentials](../architecture/k8s-runner.md#image-pull-credentials)
- [Terraform Provider](../architecture/operations/terraform-provider.md)

## Delta

All container images (agent, init_image, MCP, hook) are pulled from public registries only. No mechanism exists to authenticate with private registries.

Specifically missing:

- **Secrets proto** (`agynio/api`): `ImagePullSecret` resource messages, CRUD RPCs, `ResolveImagePullSecret` RPC.
- **Agents proto** (`agynio/api`): `ImagePullSecretAttachment` resource messages and CRUD RPCs.
- **Runner proto** (`agynio/api`): `ImagePullCredential` message and `repeated ImagePullCredential image_pull_credentials` field on `StartWorkloadRequest`.
- **Secrets service** (`agynio/secrets`): Image Pull Secret CRUD and resolution. Local value encryption/decryption for both secrets and image pull secrets.
- **Agents service** (`agynio/agents`): ImagePullSecretAttachment CRUD.
- **Agents Orchestrator** (`agynio/agents-orchestrator`): Collection of ImagePullSecretAttachments, resolution of credentials via Secrets service, registry hostname conflict detection, credential passing in `StartWorkload`.
- **k8s-runner** (`agynio/k8s-runner`): Kubernetes Secret creation from image pull credentials, Pod `imagePullSecrets` configuration, cleanup on workload stop.
- **Terraform provider** (`agynio/terraform-provider-agyn`): `agyn_image_pull_secret` and `agyn_image_pull_secret_attachment` resources.

## Acceptance Signal

- An agent with `image` referencing a private registry (e.g., `ghcr.io/my-org/my-agent:latest`) starts successfully when an image pull secret with matching credentials is attached.
- An MCP or hook referencing a private registry image starts successfully when an image pull secret is attached.
- Workload start fails with a clear error when two different image pull secrets target the same registry hostname.
- `agyn_image_pull_secret` and `agyn_image_pull_secret_attachment` Terraform resources work end-to-end.

## Notes

- Secret value storage model (local encrypted + remote provider) applies to both Secret and ImagePullSecret independently. This is a unified pattern, not shared storage.
- Encryption key for local secrets is stored in a Kubernetes Secret mounted into the Secrets service pod.
- Secret Provider becomes optional — only needed when using remote storage mode.
