# k8s-runner

New service was introduced: [k8s-runner](../architecture/k8s-runner.md).

Kubernetes-native Runner implementation (`agynio/k8s-runner`, Go). Implements the same `RunnerService` gRPC API as docker-runner, backed by the Kubernetes API (Pod, PVC, emptyDir, exec, log streaming). Embeds the OpenZiti Go SDK to bind the `runner` service. Identity provisioned by Terraform, loaded from Kubernetes Secret.

Repository `agynio/k8s-runner` does not exist yet.
