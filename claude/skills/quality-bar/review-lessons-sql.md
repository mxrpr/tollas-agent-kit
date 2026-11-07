# Review lessons: SQL, databases and migrations

Part of the review-lessons catalog (see "Catalog and caps" in section 8 of `SKILL.md`). Tag = highest severity seen: [H] High, [M] Medium, [L] Low.

## 1. SQL and queries

- [H] Never sort, window or DISTINCT rows carrying a large JSON column: sort narrow keys, extract needed fields after. Measure with realistic payload sizes.
- [M] Push date filters into the scan itself (semi-join on index), not only a window aggregate; EXPLAIN-test each common search shape.
- [M] Batch writes are one set-based statement (`UNNEST`/`push_values`); chunked statements re-scanning in one transaction add cost and release no locks.
- [M] Upserts carry a change predicate (`WHERE existing IS DISTINCT FROM EXCLUDED`); a retrying backfill writes only what is new or changed.
- [M] Don't load every candidate row into memory for a repair; stream or aggregate in SQL. Extract per page; keep only a seen-set for dedup.
- [L] `$1 IS NULL OR x = ANY($1)` defeats indexes under generic plans; `ORDER BY CASE` plus `OFFSET` sorts the whole set per page. Split queries or keyset-page.
- [L] Existence checks select only ids, never full rows with PII.
- [M] Multi-table state-changing deletes run in one transaction.
- [M] App code must not depend on deploy order: SQL that works before and after the migration when it may not have run.
- [M] A race fix is proven against the in-progress case (concurrent transaction mid-delete), not only the committed case: plain `EXISTS` sees an uncommitted delete; `FOR KEY SHARE` waits it out.
- [M] Lock rows across tables in one global order; same-table rows `ORDER BY id`.
- [M] Rewriting a DB URL keeps its query string (`sslmode`); use the driver's options API, not string splicing.
- [M] Tests run migrations, and compare upgraded vs fresh schema; schema definitions kept in several places are checked for consistency.

## 2. Migrations

- [M] Migration squashes don't rely on `CREATE TABLE IF NOT EXISTS`; verify the squashed files never shipped, or repair/refuse partial states.
- [M] Replayed-on-boot migrations are idempotent and cheap (early exit when done). Runners fail fast (`ON_ERROR_STOP`).
- [M] Destructive or unique-index migrations have a preflight for violating data, with statement and lock timeouts, checking the right DB.
- [M] No `CREATE INDEX` without `CONCURRENTLY` on large hot tables; confirm by EXPLAIN that a query needs it.
- [M] Re-check migration numbers for collisions with the target branch.
- [L] Verify index drops are covered (leading columns, direction, partiality); `lock_timeout` on live DDL.
- [L] Old migrations referencing dropped columns are guarded; one-release shims get a removal note; headers stay accurate.
- [L] Standard SQL over vendor-specific forms where one exists; no `GROUP BY` output alias.
- [L] Day boundaries compared in explicit UTC on both bounds; test under a non-UTC TZ.
- [L] Multi-column rows read into named structs, not same-typed positional tuples.

## 3. SQL tests

- [H] Destructive set-based deletes: several rows whose paired columns all differ, so misalignment fails.
- [M] DB-backed tests skip loudly or fail through one shared helper; never `Err(_) => return`, never a dev DB fallback.
