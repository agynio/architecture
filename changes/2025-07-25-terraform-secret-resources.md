# Terraform Provider — Secret Provider and Secret Resources

**Date:** 2025-07-25

## Target

- [Architecture: Terraform Provider](../architecture/operations/terraform-provider.md)

## Delta

The following are not implemented in the Terraform provider:

- `agyn_secret_provider` resource (CRUD via `CreateSecretProvider`, `GetSecretProvider`, `UpdateSecretProvider`, `DeleteSecretProvider`)
- `agyn_secret` resource (CRUD via `CreateSecret`, `GetSecret`, `UpdateSecret`, `DeleteSecret`)

The `SecretsGateway` client already exists in the provider (used by `agyn_image_pull_secret`). The Gateway proto methods already exist for secret providers and secrets. No API-side work is needed — only Terraform provider implementation.

## Acceptance Signal

- `agyn_secret_provider` creates a Vault provider, reads it back, updates address/token, and deletes it.
- `agyn_secret` creates a local secret, reads it back, updates name and value, and deletes it.
- `agyn_secret` creates a remote secret referencing an `agyn_secret_provider`, reads it back, and deletes it.
- `agyn_env` with `secret_id` referencing an `agyn_secret` provisions a secret-backed environment variable end-to-end.
