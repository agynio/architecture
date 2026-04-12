# Database Migrations

Rules for SQL migration files across all services that use the filename-based migration runner (`internal/db/migrate.go`).

## Migration Runner

Each service embeds its SQL migrations in a `migrations/` directory. The runner applies files in lexicographic order, tracking each applied file by **filename** in a `schema_migrations` table:

```sql
CREATE TABLE IF NOT EXISTS schema_migrations (
    version    TEXT PRIMARY KEY,
    applied_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

The runner has no content hashing — it uses the filename as the sole version key.

## Rules

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

Schema migrations (`ALTER TABLE`, `CREATE INDEX`, etc.) must not contain `INSERT`, `UPDATE`, or `DELETE` statements against application tables. Seed data belongs in a separate mechanism.

### 5. Backward Compatibility Window

A migration must not break the previous service version that is still running during a rolling deployment. Destructive changes (drop column, drop table) require a two-phase approach:

1. **Phase 1**: Deploy a version that stops reading/writing the column.
2. **Phase 2**: Add the migration that drops the column.

## Violations and Recovery

If a migration file has already been mutated or renamed in a released version:

1. **Do not revert the file to its original state** — databases running the current version already applied the mutated content.
2. Add a new migration that makes the schema converge regardless of which version of the mutated migration was applied. Use `IF NOT EXISTS` / `IF EXISTS` guards.
3. Document the incident in the service's `CONTRIBUTING.md`.
