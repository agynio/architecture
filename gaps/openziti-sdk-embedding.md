# OpenZiti SDK Embedding

Orchestrator, Runner, and Gateway now use the embedded OpenZiti Go SDK instead of an OpenZiti sidecar or tunneler. This avoids routing conflicts with the Istio sidecar.

## What changed

- **Orchestrator** dials runners via `zitiContext.Dial("runner")` over the OpenZiti overlay. No Istio-based Runner connectivity.
- **Runner** binds the `runner` OpenZiti service via `zitiContext.Listen("runner")` and accepts gRPC connections through it.
- **Gateway** binds the `gateway` OpenZiti service via `zitiContext.ListenWithOptions("gateway", ...)`.
- **Orchestrator → Runner** is always OpenZiti, regardless of whether the runner is internal or external. No protocol branching.
- **Internal runner identity** is provisioned by Terraform at deployment time — certificate and key stored as Kubernetes Secret, mounted into the pod.
- **Orchestrator identity** is provisioned the same way (Terraform, Kubernetes Secret).

See [Authentication — SDK Embedding](../architecture/authn.md#sdk-embedding) and [OpenZiti Integration — Runner Provisioning](../architecture/openziti.md#runner-provisioning).
