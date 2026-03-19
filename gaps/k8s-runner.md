# k8s-runner

New service. Does not exist yet.

## Repository

`agynio/k8s-runner` — to be created.

## What to implement

Full [k8s-runner architecture](../architecture/k8s-runner.md):

- Go service implementing the `RunnerService` gRPC API
- Kubernetes API backend: Pod, PVC, emptyDir, exec, log streaming
- OpenZiti Go SDK embedded — bind `runner` service, accept gRPC connections
- Identity loaded from Kubernetes Secret (provisioned by Terraform)
- RBAC: ServiceAccount with Role scoped to workload namespace
- `restartPolicy: Never` on all Pods
- Helm chart for deployment
