# Cluster Admin Authorization

## Target

- [Authorization — Cluster Permissions](../architecture/authz.md#cluster-permissions)

## Delta

The authorization model has no `cluster` type or `admin` relation. No mechanism for cluster-level administrative operations. No bootstrap process for seeding the initial admin user and API token.

## Acceptance Signal

- OpenFGA model includes `cluster` type with `admin` relation.
- Terraform bootstrap creates the initial admin user, API token, identity registration, and `cluster:admin` tuple.
- Services can check `cluster:admin` permission for cluster-level operations.

## Notes

- Required by: Apps Service, future Runners Service.
- Bootstrap writes directly to Users service PostgreSQL and OpenFGA.
