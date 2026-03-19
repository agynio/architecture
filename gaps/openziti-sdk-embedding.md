# OpenZiti SDK Embedding

Architecture now specifies that Orchestrator, Runner, and Gateway embed the OpenZiti Go SDK instead of using an OpenZiti sidecar/tunneler.

## Affected services

### agynio/agents-orchestrator

- Embed OpenZiti Go SDK
- Dial `runner` service via `zitiContext.Dial("runner")` for all Runner gRPC calls
- Load OpenZiti identity from Kubernetes Secret
- Remove any Istio-based Runner connectivity

### agynio/gateway

- Confirm Gateway already uses SDK embedding for `zitiContext.ListenWithOptions("gateway", ...)`
- If using sidecar/tunneler, migrate to embedded SDK

### agynio/k8s-runner

- New service — implement with SDK from the start (see [k8s-runner gap](k8s-runner.md))

## Infrastructure (agynio/bootstrap)

- Terraform: create OpenZiti identity for internal Runner, enroll, store as Kubernetes Secret
- Terraform: create OpenZiti identity for Orchestrator (if not already done), store as Kubernetes Secret
- Helm charts: mount identity Secrets into Orchestrator and Runner pods
