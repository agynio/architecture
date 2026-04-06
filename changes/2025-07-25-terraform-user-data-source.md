# Terraform Provider — User Resource and Data Source

**Date:** 2025-07-25

## Target

- [Architecture: Terraform Provider](../architecture/operations/terraform-provider.md)
- [Architecture: Users](../architecture/users.md)
- [Architecture: Gateway](../architecture/gateway.md)

## Delta

The following are not implemented:

- Users service: `GetUserByOIDCSubject` method (admin, cluster admin authorization)
- Gateway: `GetUserByOIDCSubject` added to `UsersGateway`
- Gateway proto: `GetUserByOIDCSubjectRequest` / `GetUserByOIDCSubjectResponse` messages
- Terraform provider: `agyn_user` resource (CRUD via `CreateUser`, `GetUser`, `UpdateUser`, `DeleteUser`)
- Terraform provider: `data.agyn_user` data source (read via `GetUserByOIDCSubject`)
- Terraform provider: `usersGateway` client added to provider client
- Terraform provider: `agynio/api/users/v1/users.proto` and `agynio/api/gateway/v1/users.proto` added to `buf.gen.yaml`

## Acceptance Signal

- `data.agyn_user` resolves an OIDC-provisioned user by subject and returns `identity_id`.
- `agyn_user` creates a user, reads it back, updates profile fields and cluster role, and deletes it.
- `data.agyn_user` combined with `agyn_membership` provisions organization membership for an existing user.
