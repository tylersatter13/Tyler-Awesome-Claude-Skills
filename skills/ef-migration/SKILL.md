---
name: ef-migration
description: Plan and create safe EF Core schema changes - entity configuration, migrations, expand-and-contract for breaking changes, data backfills, and rollback. Use when a task adds or changes tables, columns, indexes, or relationships, or when asked to create or review an EF Core migration.
---

# ef-migration

Part of the `dotnet-flow` suite (see `dev-flow`). Runs in the **Design** phase, and again in
**Build** when the migration is created. Its question: *can this schema change be deployed and
rolled back without downtime or data loss?*

## When to use

- The spec or `design.md` needs new or changed tables, columns, indexes, or constraints.
- `build-slice` hits a schema change mid-slice.
- Reviewing a migration someone else wrote.

## Reads

- `.work/<slug>/spec.md`, `design.md`, `decisions.md`.
- The `DbContext`, the `IEntityTypeConfiguration<T>` classes, and the latest migrations and model
  snapshot. Check how the repo applies migrations (startup, a bundle, or a pipeline script).
- Conventions (via `dev-flow`).

## Classify the change

| Change | Safe in one deploy? |
|---|---|
| New table, new nullable column, new index on a small table | Yes |
| New non-null column | Only with a default value, or a backfill first |
| Index on a large table | Yes if created `ONLINE` (SQL Server Enterprise / Azure SQL), otherwise plan a window |
| Rename column or table | No: expand and contract |
| Change column type or shrink length | No: expand and contract |
| Drop column or table | No: stop using it first, drop in a later release |

## Expand and contract

For breaking changes, split across releases so old and new code both work during a deploy:

1. **Expand**: add the new column or table. Code writes to both and reads from the old.
2. **Migrate**: backfill existing rows in batches, idempotently.
3. **Switch**: code reads from the new one.
4. **Contract** (a later release): stop writing the old one, then drop it.

Record the plan and which release each step ships in to `design.md`.

## Steps

1. Classify the change with the table above and write the plan into the Data changes section of
   `design.md`.
2. Update the entity and its `IEntityTypeConfiguration<T>`: explicit column types, lengths,
   `decimal` precision, required/optional, indexes, and the concurrency token if used.
3. Create the migration with `dotnet ef migrations add <Name>` with a name describing the change
   (`AddOrderExternalReference`), using `--project` and `--startup-project` as the repo does.
4. **Read the generated migration.** Watch for drops you didn't intend, a rename generated as
   drop plus add (data loss), and missing defaults on non-null columns. Fix the configuration and
   regenerate rather than hand-editing, except for data backfills.
5. Generate the SQL with `dotnet ef migrations script --idempotent` and skim it the way a DBA would.
6. Check the rollback: `Down()` works, or the rollback plan is documented (for example, the
   expand step is harmless to leave in place).
7. Test it: the integration tests run migrations against a Testcontainers SQL Server (see
   `dotnet-test`), so a broken migration fails the suite.

## Exit criteria

- The migration matches the plan, with no unintended drops or data loss.
- Idempotent SQL has been reviewed, and rollback is either `Down()` or a written plan.
- Tests pass against a real SQL Server with the migration applied.

## Writes

- Entity, configuration, and migration files in the repo.
- `.work/<slug>/design.md`: Data changes, with release steps for expand and contract.
- `.work/<slug>/decisions.md`: anything deferred to a later release (like the contract step).

## Next skill

`build-slice`.
