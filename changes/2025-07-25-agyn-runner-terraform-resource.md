# 2025-07-25: `agyn_runner` Terraform Resource

## Target

- [Terraform Provider](../architecture/operations/terraform-provider.md)
- [Runners — Terraform Provisioning](../architecture/runners.md#terraform-provisioning)

## Delta

The `agyn_runner` resource is not implemented in the [Terraform provider](https://github.com/agynio/terraform-provider-agyn). The provider has no runner-related code — no `RunnersGateway` client, no resource definition, no acceptance tests.

Specifically missing:
- `RunnersGateway` ConnectRPC client in the provider's `Client` struct (alongside existing `AgentsGateway` and `AppsGateway`)
- `agyn_runner` resource implementation (`internal/resources/runner_resource.go`)
- Registration in the provider's `Resources()` method
- Acceptance tests (`internal/provider/resource_runner_test.go`)

## Acceptance Signal

- `agyn_runner` resource is implemented, registered, and passing acceptance tests.
- A runner can be provisioned via `terraform apply`, its `service_token` captured, and used to enroll a runner.

## Notes

The proto API is already defined — `RunnersGateway` in `agynio/api/proto/agynio/api/gateway/v1/runners.proto` exposes all required RPCs. The implementation follows the same pattern as `agyn_app`.
