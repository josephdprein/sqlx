# sqlx-cli UX Notes

Working notes for the `claude/sqlx-cli-ux-improvements` branch.
High-level goal: make sqlx-cli the ideal command-line migration runner.

## Confirmed UX gaps in current sqlx-cli

### 1. `migrate info` hides applied-but-missing migrations

`sqlx-cli/src/migrate.rs:150` iterates `migrator.iter()` (the on-disk list)
and looks up each version in `applied_migrations`. There is no second pass
over `applied_migrations` for rows that have no matching local file, so
`migrate info` prints nothing about them — yet `migrate run` will hard-fail
on exactly that case (`MigrateError::VersionMissing`, raised in
`validate_applied_migrations` at `migrate.rs:208`).

`migrate info` also does not surface the dirty-version state (a partially
applied migration whose row was left in `_sqlx_migrations` with
`success = false`). Both `run` and `revert` refuse to proceed when dirty,
but `info` does not warn.

### 2. No recovery path for stuck/edited migrations

The only override commands that exist today are:

- `--ignore-missing` on `run`/`revert` — suppresses the "applied but not in
  source" error; does nothing about checksum mismatch.
- `migrate override skip` — marks pending migrations as applied without
  running them. This is the *opposite* of what you usually want in dev:
  re-running an already-applied migration, or accepting a new checksum
  after editing the file.

Missing operations include:

- Re-stamp the checksum after a deliberate edit.
- Mark an already-applied migration as un-applied so it re-runs.
- Delete a stray DB row whose file no longer exists.
- Recover from a dirty state without manually editing
  `_sqlx_migrations`.

Today the workflow is "psql into the DB and hand-edit the migrations
table," which is exactly the workflow a CLI should obviate.

### 3. `AppliedMigration` discards useful columns

`sqlx-core/src/migrate/migration.rs:81` only carries `version` + `checksum`.
The DB stores `installed_on` and `execution_time` (see the schema in each
driver's `migrate.rs`, e.g. `sqlx-postgres/src/migrate.rs:133`), but
`list_applied_migrations` selects only the two fields it keeps. As a
result `migrate info` cannot show "applied at <date>, took <duration>" —
the first information you want when triaging.

### 4. `migrate info` exit code is always 0

Even with a mismatched checksum or a missing migration, `info` returns
success. CI cannot use it as a drift-detection step; `migrate run
--dry-run` is the only alternative and it bails on the first error
instead of reporting all of them.

### 5. Error messages are bare `thiserror` strings

When `run` fails on a checksum mismatch, the user sees
`migration 20240101000000 was previously applied but has been modified`
with no filename, no suggested next command, no hint that a recovery flag
might exist. New users hit a wall here.

Definitions live in `sqlx-core/src/migrate/error.rs`.

### 6. Smaller papercuts

- `migrate revert` prints `Applied` instead of `Reverted`
  (`sqlx-cli/src/migrate.rs:399`).
- No machine-readable output (`--json`) on any subcommand — awkward for
  scripts and IDE integrations.
- The `Override` subcommand namespace is set up for dev-recovery
  operations but only has `skip` under it.
- `migrate run` has no concept of "list everything I'm about to do
  including ordering issues" — only a one-by-one apply loop that bails on
  the first problem.

## Proposed plan (sketch, not yet decided)

Grouped smallest-first so tiers can land independently.

**Tier 1 — `migrate info` becomes the source of truth.**
- Add a "missing from source" section (versions in DB, not in files).
- Show dirty version if any.
- Show `installed_on` and `execution_time` (requires extending
  `AppliedMigration` + the three driver `list_applied_migrations` impls
  to select those columns — small, additive change).
- Add `--check` (exit non-zero on any drift) so CI can use it.
- Add `--json`.

**Tier 2 — Recovery subcommands under `migrate override`.**
Each requires confirmation unless `-y`, supports `--dry-run`, prints the
SQL it ran.
- `migrate override force-checksum <version>` — update the stored
  checksum to match the local file. The fix for "I edited a migration in
  dev."
- `migrate override mark-applied <version>` — INSERT a row without
  running SQL. Like `skip` but for one specific version.
- `migrate override mark-unapplied <version>` — DELETE a row. Lets a
  migration re-run on next `migrate run`.
- `migrate override resolve-dirty <version>` — clear the dirty state.
  Today the error message says "fix and remove row from
  `_sqlx_migrations` table" — let's do it for them.
- `migrate override rerun <version>` — DELETE the row and re-apply the
  SQL in one step. Pure dev convenience.

Trait surface: needs a small additive extension to `Migrate` in
`sqlx-core/src/migrate/migrate.rs` (`update_checksum`, `delete_row`,
maybe `clear_dirty`) and the matching impls in
`sqlx-{postgres,sqlite,mysql}/src/migrate.rs`.

**Tier 3 — Error-message hints.**
On `VersionMismatch`/`VersionMissing`/`Dirty`, print the offending file
path and suggest the matching `migrate override` command. Smallest
change, highest-impact for new users.

## Design decisions captured so far

- Destructive override commands should **confirm by default**, with `-y`
  to skip. Matches `database drop`.

## Open questions / TODO

- Pull in the issue tracker's most-upvoted CLI items to confirm priority.
- Competitive analysis: how do Flyway, Alembic, Diesel, refinery, goose,
  dbmate, golang-migrate, Atlas, Prisma Migrate handle the same gaps?
  Especially: drift detection, checksum recovery, mark-applied/unapplied,
  baseline/repair flows, machine-readable output.
