# Exercise 3 — Verify with before/after plans

**Goal:** Turn "it feels faster" into proof. For each fix from Exercise 2, produce a side-by-side before/after `EXPLAIN (ANALYZE, BUFFERS)` and a **p95** latency number from a repeatable benchmark. This is the verification half of the loop — and the skeleton of the capstone report.

**Estimated time:** ~1.5 hours.

**Prerequisite:** Exercises 1–2 complete; indexes applied.

## Step 1 — Warm-cache discipline

A single run lies (cold vs. warm cache). Establish a fair method and use it for every measurement:

1. Run the query once and **discard** it (this warms the cache).
2. Run it 3 more times; take the median of the warm runs.
3. Always capture `EXPLAIN (ANALYZE, BUFFERS)` so the plan and the timing come together.

## Step 2 — Recreate the "before" plan

You already have the "after" (indexes are in place). To capture a clean "before" for the report, you can drop the index in a transaction and roll back, so nothing is permanently lost:

```sql
BEGIN;
DROP INDEX idx_orders_cust_created;
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, status, total, created_at
FROM orders WHERE customer_id = 12345 ORDER BY created_at DESC;   -- the BEFORE plan
ROLLBACK;   -- index restored
```

> Note: `DROP INDEX` inside a transaction is fine, but you cannot use `CREATE INDEX CONCURRENTLY` inside one — for the "after" you already built the index normally, so just re-run the query outside the transaction.

Capture the "after" plan (index present) the same way, outside the transaction.

## Step 3 — Get a p95 number with pgbench

An `EXPLAIN ANALYZE` time is one run; a latency budget is about the distribution. Use `pgbench` with a custom script to hammer one query and report percentiles.

```bash
# file: q2.sql
\set cid random(1, 199999)
SELECT id, status, total, created_at
FROM orders WHERE customer_id = :cid ORDER BY created_at DESC;
```

```bash
pgbench -n -f q2.sql -T 20 -c 4 -P 5 crunch_tune
```

`pgbench` prints `latency average` and, with newer versions, more detail; for a clean p95, add `--progress` and inspect, or run with `-r` for per-statement latency. Record the average and the tail you observe. Do this for the "before" (drop index, benchmark, recreate) and "after" states.

## Step 4 — Build the before/after table

For each of your three fixes, fill in a row like this in `report.md`:

| Query | Metric | Before | After | Win |
|-------|--------|-------:|------:|----:|
| Q2 | plan node | `Seq Scan` | `Index Scan` | — |
| Q2 | rows scanned | 5,000,000 | ~25 | 200,000× |
| Q2 | buffers read | ~38,000 | ~6 | — |
| Q2 | warm exec time | ~430 ms | ~0.4 ms | ~1000× |
| Q2 | pgbench avg | ~440 ms | ~0.9 ms | — |

Repeat for Q1 and Q4. For Q3, record whatever you decided (and if you left it alone, show that it already meets budget — or does not).

## Step 5 — Write the "why"

Under the table, one paragraph per query: *why* the change worked, in plan terms. Not "I added an index," but "the `customer_id` B-tree let Postgres jump straight to the ~25 matching rows instead of scanning 5M and discarding 4,999,975 — buffers read fell from ~38k pages to ~6, so the query went from disk-bound to memory-resident." That paragraph is what makes it a *report*, not a changelog.

## Expected result

- A before/after table for each fixed query with plan node, rows, buffers, and timing.
- A p95/average latency pair per query from `pgbench`.
- A one-paragraph plan-level explanation per fix.

## Done when…

- [ ] Every claimed improvement has a **before** plan and an **after** plan, captured with `ANALYZE, BUFFERS`.
- [ ] Every improvement has a benchmark number (not just a single `EXPLAIN` run).
- [ ] The report explains *why* each fix worked in terms of the plan, not just "added index."
- [ ] Any fix that did **not** help is noted as reverted (honesty counts — a null result is a result).

## Stretch

- Reset `pg_stat_statements`, replay the workload, and confirm your former top-3 offenders have dropped out of the top of the list — proof at the system level, not just per-query.
- Measure total index size added (`pg_indexes_size('orders')` before vs. after) and note the storage cost of your wins — every optimization has a price.

## Submission

Commit the completed `report.md` under `c33-week-12/exercise-03/`. This report is a template you will reuse for the capstone.
