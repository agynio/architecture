# Database Migrations

Rules for the migrations a service applies to its own database at startup: **schema migrations**, applied by the filename-based runner (`internal/db/migrate.go`), and **data migrations**, applied by the registry runner (`internal/db/datamigrate.go`).

## Schema Migration Runner

Each service embeds its SQL migrations in a `migrations/` directory. The runner applies files in lexicographic order, tracking each applied file by **filename** in a `schema_migrations` table:

```sql
CREATE TABLE IF NOT EXISTS schema_migrations (
    version    TEXT PRIMARY KEY,
    applied_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

The runner has no content hashing — it uses the filename as the sole version key.

## Schema Migration Rules

### 1. Migrations Are Immutable

Once a migration file is committed to `main`, its **filename and contents must never change**. Any schema change — including corrections to a previous migration — must be a new file with a new sequence number.

Renaming a file (e.g., `0003_add_nickname.sql` → `0003_add_email.sql`) creates a new version key. Databases that applied the old filename will see the renamed file as unapplied and attempt to re-execute its contents, causing failures.

### 2. Migrations Must Be Idempotent

Every migration must succeed when applied to a database that already has the described state. Use defensive DDL:

| Operation | Pattern |
|-----------|---------|
| Add column | `ALTER TABLE t ADD COLUMN IF NOT EXISTS col TYPE ...` |
| Create table | `CREATE TABLE IF NOT EXISTS t (...)` |
| Create index | `CREATE INDEX IF NOT EXISTS idx ON t (...)` |
| Drop column | `ALTER TABLE t DROP COLUMN IF EXISTS col` |
| Drop table | `DROP TABLE IF EXISTS t` |
| Drop index | `DROP INDEX IF EXISTS idx` |

This protects against bootstrap re-apply and partial-failure recovery scenarios where the DDL succeeded but the `schema_migrations` insert did not commit.

### 3. Sequence Numbering

Files are named `NNNN_<description>.sql` with a zero-padded four-digit prefix. The next number is always `max(existing) + 1` on the `main` branch at the time the PR is opened.

### 4. No Data Mutations in Schema Migrations

Schema migrations (`ALTER TABLE`, `CREATE INDEX`, etc.) must not contain `INSERT`, `UPDATE`, or `DELETE` statements against application tables. Seed data and backfills belong in [Data Migrations](#data-migrations).

### 5. Backward Compatibility Window

A migration must not break the previous service version that is still running during a rolling deployment. Destructive changes (drop column, drop table) require a two-phase approach:

1. **Phase 1**: Deploy a version that stops reading/writing the column.
2. **Phase 2**: Add the migration that drops the column.

## Data Migrations

A data migration mutates application data: seed rows, a backfilled column, or records outside the service's database that its own rows imply — [authorization tuples](../authz.md#tuple-lifecycle) in particular.

### Runner

Each service declares an ordered registry of data migrations and applies it through `internal/db/datamigrate.go`, tracking each applied migration by version in a `data_migrations` table:

```sql
CREATE TABLE IF NOT EXISTS data_migrations (
    version    TEXT PRIMARY KEY,
    applied_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

A migration is a Go function, not a file: the mechanism has to reach stores the database cannot, and one registry covering both is what keeps a tuple repair and a column backfill from needing separate machinery. Migrations that touch only the service's own database execute SQL inside that function.

Versions are named `NNNN_<description>`, zero-padded to four digits, the next always `max(existing) + 1` on `main` at the time the PR is opened. A version is immutable once merged: the version string is the sole identity the ledger records, so a migration whose behavior must change is a new version.

### Execution

| Property | Rule |
|----------|------|
| Order | Ascending version. A failure stops the run; later versions do not apply |
| Exclusion | `pg_try_advisory_lock`. One replica applies the registry; a replica that cannot take the lock skips it |
| Recording | The version is recorded only after the migration returns successfully |
| Delivery | At-least-once. A crash between the work and the ledger insert re-runs the migration |
| Idempotency | Required. A migration must reach the same end state against data that already has it |
| Blocking | Never blocks serving. Data migrations run after schema migrations, alongside a service that is already accepting requests |
| Direction | Forward only |

At-least-once is a property of the mechanism rather than a choice. A data migration may write to stores outside the service's database, and no transaction spans them.

Because data migrations do not block serving, a service that is serving is **not** necessarily a service whose data migrations have completed — unlike [schema migrations](platform-installation.md), which are a precondition for accepting traffic. A data migration therefore never repairs data the current write path depends on; it converges records written before the code that would have written them correctly.

### Scope

A data migration converges a population that is finite and fully determined when it runs. Work that must repeat to keep data correct is not a migration: it is steady-state behavior, and it belongs in the service's design rather than in the registry. A registry entry that would be worth running again next month is a sign the write path is wrong.

A data migration that repairs tuples enqueues them into the [authorization outbox](../authz.md#relationship-writes) — a single set-based `INSERT … SELECT` — and owns none of the delivery. It targets only records predating the type or relation they need: a set that stops growing the moment the writing code ships.

### Cost

Migrations run against production-sized data. A migration that mutates rows works in set-based statements, or pages when it must compute per record; one round trip per record does not finish at scale. Tuple repairs inherit batching, ordering, and retry from the outbox processor.

## Violations and Recovery

If a migration file has already been mutated or renamed in a released version:

1. **Do not revert the file to its original state** — databases running the current version already applied the mutated content.
2. Add a new migration that makes the schema converge regardless of which version of the mutated migration was applied. Use `IF NOT EXISTS` / `IF EXISTS` guards.
3. Document the incident in the service's `CONTRIBUTING.md`.
