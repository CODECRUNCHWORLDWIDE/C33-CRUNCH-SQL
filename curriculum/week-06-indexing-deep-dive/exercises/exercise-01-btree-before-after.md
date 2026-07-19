# Exercise 1 — B-tree, before and after

**Goal:** Take a genuinely slow query on 2M rows, prove it's slow with the planner, add the right B-tree, and prove the win in real numbers. This is the core loop of the whole week.

**Estimated time:** 45 minutes.

## Setup

Have the `shop` database from [Lecture 1, §0](../lecture-notes/01-how-a-btree-index-works.md). Confirm the row count and that no extra index exists yet:

```sql
\c shop
SELECT count(*) FROM orders;                       -- ~2,000,000
\d orders                                           -- only the PK index should be listed
```

If you added indexes while reading the lectures, drop them so you start clean:

```sql
DROP INDEX IF EXISTS ix_orders_created_at;
DROP INDEX IF EXISTS ix_orders_cust_created;
```

## Part A — Measure the "before"

The business question: **"Show orders placed on 2023-06-15."** Capture the baseline plan.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders
WHERE created_at >= '2023-06-15' AND created_at < '2023-06-16';
```

Record, in `notes.md`:

- The top scan node (expect `Seq Scan`).
- `actual time=... rows=...`.
- The `Buffers: shared read/hit=...` line — how many 8 KB pages were touched.
- The total `Execution Time`.

## Part B — Add the index and measure the "after"

```sql
CREATE INDEX ix_orders_created_at ON orders (created_at);
ANALYZE orders;

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders
WHERE created_at >= '2023-06-15' AND created_at < '2023-06-16';
```

Record the new plan. Expect an `Index Scan` or `Bitmap Heap Scan`, far fewer buffers, and a much lower execution time. Compute the speedup (before ÷ after) and the buffer reduction.

## Part C — Watch selectivity flip the decision

Now ask a *poorly selective* question and see the planner refuse the index on purpose:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE created_at >= '2022-01-01';   -- keeps ~everything
```

Even with the index present, this should be a `Seq Scan`. In `notes.md`, explain in one sentence *why the planner is correct* to ignore the index here.

## Part D — Break sargability, watch the index die

```sql
-- non-sargable: the column is wrapped in a function
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE date(created_at) = '2023-06-15';
```

Note that this falls back to a `Seq Scan` despite the index. Then confirm the sargable form (Part A's query) uses the index. Write one sentence on why wrapping `created_at` in `date()` disables the index.

## Expected result

| Query | Plan | Buffers | Time (order of magnitude) |
|-------|------|--------:|---------------------------|
| A — one day, no index | `Seq Scan` | tens of thousands | ~100s of ms |
| B — one day, with index | `Index`/`Bitmap` Scan | dozens–hundreds | ~1 ms |
| C — almost all rows | `Seq Scan` (index ignored, correctly) | tens of thousands | ~100s of ms |
| D — `date()` wrap | `Seq Scan` (index unusable) | tens of thousands | ~100s of ms |

Exact numbers vary by machine; the *shape* (A and B differ by 10–100×; C and D stay slow) is what matters.

## Done when…

- [ ] `notes.md` contains the **before** and **after** plans for Part A/B with buffer counts and a computed speedup.
- [ ] You can state, in one sentence, why Part C is a correct `Seq Scan` (selectivity).
- [ ] You can state, in one sentence, why Part D can't use the index (sargability).
- [ ] You dropped any pre-existing indexes before starting, so the baseline was honest.

## Stretch

- On an SSD, run `SET random_page_cost = 1.1;` then re-run Part C. Does the plan change? Explain.
- Add `ORDER BY created_at DESC LIMIT 10` to Part A's query. Does the index remove a `Sort` node? Check for `Backward` in the scan.
- Compare `EXPLAIN` (estimate only) with `EXPLAIN (ANALYZE)` (actual). Where do the estimated and actual row counts diverge, and what would you `ANALYZE` to fix it?

## Submission

Commit `notes.md` (both plans + your one-sentence answers) to your portfolio under `c33-week-06/exercise-01/`.
