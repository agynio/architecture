# Data Migrations

## Target

- [Database Migrations — Data Migrations](../architecture/operations/database-migrations.md#data-migrations)
- [Authorization — Tuple Lifecycle](../architecture/authz.md#tuple-lifecycle)

## Delta

- No service implements `internal/db/datamigrate.go`. No `data_migrations` table exists. Nothing records that a one-shot data mutation has been applied, so nothing distinguishes a repair that has run from one that has not.
- The Agents service performs its one data repair — base authorization tuples for environments predating the environment type — as a routine invoked on every process start rather than as a recorded migration. Under the mechanism it is registry entry `0001_environment_authorization`: a one-shot enqueue into the [authorization outbox](2026-08-23-authorization-outbox.md), applied once, recorded, never run again.

## Acceptance Signal

- `internal/db/datamigrate.go` and the `data_migrations` table exist in the Agents service, and the environment tuple repair is registry entry `0001_environment_authorization`.
- Restarting a service applies no data migration whose version is already recorded. A service that has applied its registry logs no repair work on subsequent starts.

## Notes

- Depends on the [authorization outbox](2026-08-23-authorization-outbox.md) landing in Agents first: the migration enqueues into it.
- The advisory lock is inert while every chart ships `replicaCount: 1`, and required before any service running data migrations scales past one replica.
- The Tuple Lifecycle table did not list environment creation, although the `environment` type section specified the tuples; the rows were added alongside this change.
