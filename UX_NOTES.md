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

## Issue-tracker research (launchbadge/sqlx, open issues)

Top-relevance open issues directly motivating our plan:

- **#3794** — "previously applied but has been modified" pain; user wants
  `--no-checksum`/`--fix-checksum` on `migrate run`. Direct precedent for
  `override force-checksum`.
- **#911** — feature request for a `baseline` command (fake-apply up to a
  version, for adopting sqlx on an existing DB). Tagged
  `cli`/`proposal`/`E-medium`. Matches `override mark-applied`.
- **#2238** — "applied but missing" error is opaque; user read sqlx
  source to understand. Motivates `mark-unapplied` + error hints.
- **#3893** — wants auto-revert-and-rerun on change. Overlaps `rerun`.
- **#1933** — `migrate info` emits misleading "wrong checksum" warnings
  for `.down.sql` files. Real bug in current info reporting, separate
  from our planned additions.
- **#3909** — comment-insensitive checksums (soft checksum semantics
  behind a flag).
- **#4231** — `revert --target-version` semantics inconsistent with
  `run`. Pure correctness issue.
- **#3972** — CLI should warn about CRLF/LF on first migration; line
  endings silently break checksums.
- **#1189** — `VersionMissing` when several SQLx-based libraries share a
  DB; only workaround is `ignore_missing`.
- **#3023, #4021** — `cargo sqlx prepare` error guidance is
  circular/misleading. Precedent for "better error hints" tier.
- **#4059** — `Migrator::run` should return count/list of executed
  migrations. Aligns with `--json` output.

Recurring closed-as-duplicate themes: "applied but missing" cluster
(#2079, #2238, #1189, #870), modified-checksum cluster (#3794, #3909,
#1933, #870). 4+ years of recurrence — these aren't fringe complaints.

Existing work to build on:

- **PR #3846 (merged)** — introduced the `migrate override` subcommand
  namespace with `skip`. Our new override subcommands slot in next to it.
- **PR #4076 (draft)** — `sqlx revalidate` for the `.sqlx/` query cache;
  unrelated to migration state.
- No CHANGELOG entry exists yet for `--check`, `--json`, repair-style
  commands, or improved `migrate info` reporting.

## Competitive analysis (other migration runners)

**Rust/Go peers**

- *Diesel CLI, refinery* — bare-minimum; same tier sqlx-cli is in today.
- *golang-migrate* — `force V` is the canonical dirty-state escape hatch
  others copied; otherwise minimal.
- *goose* — `status` shows applied-at timestamps; `validate` is a
  no-side-effects parse; `-allow-missing` for out-of-order; `fix`
  renumbers timestamps before release.
- *dbmate* — `status --exit-code` is the cleanest CI primitive seen.
  Auto-maintains a `schema.sql` snapshot — feature sqlx lacks.
- *Atlas (ariga)* — best in class. `migrate status` flags `OUT_OF_ORDER`
  and checksum mismatch as distinct states; `--format '{{ json . }}'`
  for machine output; `migrate set <version>` edits the revisions table
  (mark applied/unapplied/repair); `migrate hash` re-stamps the
  checksum file; `migrate lint` is a separate verb with 50+ analyzers.
  Error messages name the file and print the exact next command.

**Python/Ruby/JS peers**

- *Flyway* — `info` enumerates every state (`Pending`, `Success`,
  `Failed`, `Missing`, `Future`, `Ignored`, `Out of Order`, `Baseline`),
  with installed-on, installed-by, execution time. JSON via
  `-outputType=json`. `repair` realigns checksums + clears failed rows.
  `baseline` stamps an existing DB. `validate` is the CI gate.
- *Liquibase* — `clear-checksums` (bulk re-stamp),
  `mark-next-changeset-ran` (per-item fake-apply), `release-locks` for
  stuck locks, `changelog-sync` for full baseline. Checksum errors name
  the file and point to `clear-checksums`.
- *Alembic* — no checksums (so no checksum repair needed); `stamp <rev>`
  marks applied without running. `--sql` offline mode emits SQL for
  code review. `check` detects drift between models and migrations.
- *Django* — `showmigrations` with `[X]/[ ]`. `migrate --fake` /
  `--fake-initial` for stamping. `migrate --check` exits non-zero on
  pending — the canonical CI gate. `sqlmigrate` prints SQL for review.
- *Rails* — `db:migrate:status` prints `********** NO FILE **********`
  next to versions whose file is missing. Gold standard label for the
  "applied but missing" state. `db:migrate:redo STEP=N`, schema dump as
  drift anchor.
- *Prisma Migrate* — `migrate status` reports drift with copy-paste fix
  commands. `migrate resolve --applied <name>` / `--rolled-back <name>`
  is the canonical per-migration repair. `migrate diff --from-X
  --to-Y --script` emits SQL between any two schema states.
- *Knex* — `migrate:unlock` as a first-class command for stuck locks.

**Synthesis — patterns nearly every leader shares**

1. `info`/`status` enumerates *every* state including "applied but
   missing locally" as a distinct, named state. Rails' `NO FILE` and
   Flyway's `Missing` are the gold standard label.
2. A first-class re-stamp/repair command exists (`flyway repair`, Atlas
   `migrate hash`+`migrate set`, Liquibase `clear-checksums`, Prisma
   `migrate resolve`, golang-migrate `force`). sqlx forcing users into
   psql is the outlier.
3. `--check` exit code + JSON/machine output is table stakes for CI.
   dbmate `status --exit-code`, Atlas `--format json`, Django
   `migrate --check`, Prisma `migrate status` exit codes.
4. Error messages name the offending file and the next command to run.
   Atlas's "run `atlas migrate hash`" and Liquibase's pointer to
   `clear-checksums` are the gold standard.
5. Lint/validate as a verb separate from `run` (goose `validate`, Atlas
   `migrate lint`, Flyway `validate`) — decouples "is this safe?" from
   "apply it."

**Adjacent features worth considering** (not in original plan):

- Offline SQL emission (Alembic `--sql`, Prisma `migrate diff --script`,
  Django `sqlmigrate`) for code review and managed-DB workflows.
- Schema-snapshot file maintained alongside migrations (Rails
  `schema.rb`, dbmate `schema.sql`) as a drift anchor.
- `migrate unlock` for releasing a stuck advisory lock (Knex pattern).
- Documented exit-code contract (0 = in sync, 1 = pending, 2 = drift,
  3 = dirty) so CI scripts can branch on cause.
