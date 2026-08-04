# Image Pull Proxy

## Target

- [Image Proxy](../architecture/image-proxy.md)
- [Images — How Images Are Pulled](../product/images/images.md#how-images-are-pulled)
- [Agents Orchestrator — Workload Spec Assembly](../architecture/agents-orchestrator.md)
- [k8s-runner — Image Pull Credentials](../architecture/k8s-runner.md#image-pull-credentials)

## Delta

### Image Proxy

The component does not exist. No repository, no chart, no deployment, no address for a runner to pull from.

- No OCI Distribution surface, so nothing serves a pull.
- No reference scheme. `<proxy-host>/<org-slug>/<image-name>:<tag>` resolves to nothing, and no image reference is rewritten anywhere.
- No pull credential lifecycle: `MintPullCredential` and its revocation counterpart do not exist, nothing scopes a credential to a workload and a set of images, and no TTL backstop exists for a credential whose workload never started.
- No upstream authentication path, so the credential stored on an [Image](../architecture/resource-definitions.md#image) has no consumer.
- No blob cache. Every pull would reach upstream, on the cold-start path of every workload.

### Agents Orchestrator

- Assembly resolves image pull secret attachments across the environment, agent, MCPs and hooks, resolves their credentials through the Secrets service, and merges them with conflict detection. All of it goes.
- Nothing rewrites image references to the proxy, and nothing mints or revokes a per-workload pull credential.

### k8s-runner

- The `dockerconfigjson` Secret it builds carries an upstream registry hostname, username and password. It carries the proxy host and the workload's minted credential instead — same mechanism, different contents, and one Secret per workload rather than one per credential.
- Workload Pods do not set `automountServiceAccountToken: false`, so a workload with API access could read its neighbours' pull credentials in the shared workload namespace.
- Nothing prevents image pull secrets being attached to the workload ServiceAccount, which the kubelet would merge into every workload.

### Agents Service

`ImagePullSecretAttachment` still exists, with agent, MCP, hook and environment targets, its CRUD surface, and its authorization entries.

### Secrets Service

The `ImagePullSecret` resource and its resolution endpoint still exist.

### Organizations Service

The organization model has no `slug` — only `id`, `name`, and the sandbox defaults. The proxy reference scheme needs one, and [Apps](../architecture/apps.md#identification) already documents a `{org-slug}/{app-slug}` address built on the same missing field. Cluster-wide uniqueness, and a Console create form that collects it, do not exist either.

### Local Bundle

The bake pre-pulls every image under its upstream name. Content is keyed by the reference it was pulled with, so catalog images — which workloads will request as proxy references — miss the cache on first start and attempt a real pull. Platform containers are unaffected, since they keep their upstream references.

## Acceptance Signal

- A workload spec contains no upstream registry address for any image a user configured, and no upstream credential. Platform containers — `agynd-cli-init`, `agyn-cli-init` and the Ziti sidecar — remain chart-pinned and are deliberately not proxied, since the proxy cannot serve components a workload needs before the proxy is reachable.
- The `dockerconfigjson` Secret in the workload namespace contains only the proxy host and a credential that is useless once the workload stops.
- A workload can pull the images its environment and MCPs name, and cannot pull an image it was not granted, using the same credential.
- A `public` image owned by another organization pulls successfully without that organization's credential existing anywhere in the consuming cluster.
- A booted local bundle VM starts a workload with no network access to any registry.
- No `ImagePullSecret` or `ImagePullSecretAttachment` remains in any API, database, Console surface, or Terraform resource.

## Notes

- Organizations gain a cluster-wide unique `slug`, which the reference scheme needs and which retroactively grounds the app address scheme. It is mutable; a rename changes app addresses and image references, costing a one-time node cache miss and breaking nothing, since neither consumer stores a resolved reference.
- Reachability is the deployment constraint: image pulls are performed by the node's container runtime, not the workload, so they cannot be intercepted by the pod's Ziti sidecar. The proxy is reached over ordinary networking at an address every runner's nodes resolve and trust. A per-cluster caching proxy dialling the central one over OpenZiti is described as an option, not a requirement, and is not part of this change.
- The proxy is on the cold-start path for every workload. A pull-through cache is part of the design rather than an optimization.
- Hooks are removed separately ([Remove Hooks](2026-08-03-remove-hooks.md)); until they are, they are the one consumer of free-form image references that this change does not cover.
- Depends on [Image Catalog](2026-08-03-image-catalog.md) for the records and credentials it resolves.
