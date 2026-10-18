# ADR 0004: Database migration strategy

- Status: Proposed (needs director approval)
- Date: 2026-10-17
- Related: docs/spec.md "Engineering standards", ADR 0003

## Decision in one line

Database changes are made with **Alembic** in three safe steps: **expand** (add new things alongside
old ones), **migrate** (move the code and copy the data over in small batches), **contract** (remove
the old things in a later release). Every change has a tested "undo" step. Large data copies run as
background jobs in small batches, never inside the change script itself.

Plain-English terms:
- **Migration**: a versioned script that changes the database structure.
- **Reversible**: each migration has an `upgrade` and a `downgrade` that undoes it.
- **Backfill**: filling a new column or table from existing data.
- **Batched**: doing the backfill a few thousand rows at a time so the site stays fast and the job can
  stop and resume.

## Context

- CLAUDE.md: "expand -> migrate -> contract; every migration reversible; backfills as batched jobs."
- The API and workers must keep running while the database changes (old and new code may run side by
  side for a while during a deploy).
- Telemetry tables are large and partitioned by day; locking them for long would stall ingest.
- Several tickets need schema; parallel tickets must not create competing migration "heads".

## Options considered

| Option | Pros | Cons |
|---|---|---|
| **Alembic, hand-reviewed, expand/migrate/contract** (chosen) | Standard with SQLAlchemy 2; supports downgrade; autogenerate as a starting point | Needs discipline; autogenerate misses RLS policies and partitions, so those are written by hand |
| Raw SQL files with a runner (e.g. Flyway-style) | Very explicit | Second toolchain; no model diffing; Java dependency for Flyway |
| Apply schema from models on startup (`create_all`) | Fast to start | No history, no downgrade, unsafe in production |

## Decision

1. **Tool**: Alembic in `core/migrations/`, run only as the `cp_owner` role via `make migrate`
   (part of `make up`). App roles cannot change schema.
2. **Three phases, separate releases**:
   - Expand: add tables, nullable columns, new indexes (`CREATE INDEX CONCURRENTLY`), new RLS policies.
     Never drop or rename in an expand migration.
   - Migrate: code writes both old and new shapes; a batched backfill job copies data.
   - Contract: only after no running code reads the old shape, drop it in a later migration.
3. **Every migration has a working `downgrade`**. CI runs upgrade -> downgrade -> upgrade on a seeded
   database for every migration (`test_all_migrations_upgrade_downgrade_roundtrip`). If a downgrade
   would destroy data (contract steps), it must recreate the structure and the PR explains the data
   impact; such PRs are **risk:high**.
4. **Lint rule**: migrations whose filename is not tagged `contract_` may not contain `drop_table`,
   `drop_column`, `alter_column(... nullable=False)` on existing columns, or renames
   (`test_no_destructive_ops_outside_contract_migrations`).
5. **Lock safety**: each migration sets `lock_timeout = '5s'` and `statement_timeout = '60s'` so a
   blocked migration fails fast instead of freezing the API.
6. **Backfills** are worker jobs in `worker/src/cp_worker/jobs/`, processing **5,000 rows per batch**
   in keyset order, committing per batch, recording progress in a `job_checkpoints` table so a restart
   resumes where it stopped, and sleeping between batches to cap load
   (`test_backfill_is_batched_and_resumable`).
7. **One migration per wave**: only one ticket per delivery wave may add an Alembic revision, so there
   is always a single, linear chain of migrations (`test_single_alembic_head`).
8. **Partitions**: telemetry partitions (one per day) are created ahead of time by a scheduled job
   (14 days ahead) and old ones dropped by the retention job, not by migrations.

## Consequences

- Good: zero-downtime changes; every change can be undone; parallel work cannot fork the migration chain.
- Cost: renames and removals take two releases instead of one.
- Cost: RLS policies, partitions and pgvector indexes are written by hand and reviewed carefully.
- Every migration PR is **risk:high** (CLAUDE.md) and shows upgrade and downgrade output.
