# Exercise 3 — Composite, partial, and covering indexes

**Goal:** Get column order right in a composite index, shrink an index with a partial `WHERE`, and trigger an `Index Only Scan` with `INCLUDE`. Three ideas, one query pattern each, all measured.

**Estimated time:** 45 minutes.

## Setup

`shop` database, `orders` table. Clear the slate of ad-hoc indexes so results are clean (keep the PK):

```sql
\c shop
DROP INDEX IF EXISTS ix_orders_created_at;
DROP INDEX IF EXISTS ix_orders_cust_created;
DROP INDEX IF EXISTS ix_orders_cover;
DROP INDEX IF EXISTS ix_orders_pending;
```

## Part A — Composite column order matters

The query pattern: **"a single customer's recent orders, newest first."**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders
WHERE customer_id = 42 AND created_at >= '2023-01-01'
ORDER BY created_at DESC;
```

Build the *wrong* order first (range column first), test, then the *right* order:

```sql
-- wrong: range column leads
CREATE INDEX ix_wrong ON orders (created_at, customer_id);
ANALYZE orders;
-- (run the query above, record the plan)

DROP INDEX ix_wrong;

-- right: equality column leads, sort column second
CREATE INDEX ix_orders_cust_created ON orders (customer_id, created_at);
ANALYZE orders;
-- (run the query above, record the plan)
```

Compare. The right index should give an `Index Scan` with **no `Sort` node** (the index already orders by `created_at` within a customer). Note in `notes.md` which index the planner preferred and whether a `Sort` appeared.

## Part B — Partial index: smaller and cheaper

Most orders are not `pending`, but you query pending ones constantly. Build a full index and a partial one and compare sizes:

```sql
CREATE INDEX ix_full_status ON orders (status, created_at);
CREATE INDEX ix_orders_pending ON orders (created_at) WHERE status = 'pending';
ANALYZE orders;

SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE relname = 'orders' AND indexrelname IN ('ix_full_status','ix_orders_pending');
```

Then confirm the partial index is used when the query implies its predicate:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders
WHERE status = 'pending' AND created_at >= '2023-06-01';
```

Record: the size difference, and that the plan uses `ix_orders_pending`. Then run a query for `status = 'paid'` and confirm the partial index is **not** used (its `WHERE` doesn't match).

## Part C — Covering index → index-only scan

The query: **"for one customer, list order dates and totals"** — only three columns needed.

```sql
-- before: the composite index still needs a heap fetch for total_cents
EXPLAIN (ANALYZE, BUFFERS)
SELECT created_at, total_cents FROM orders WHERE customer_id = 42;

-- covering index: carry total_cents as a payload column
CREATE INDEX ix_orders_cover
ON orders (customer_id, created_at) INCLUDE (total_cents);
ANALYZE orders;

EXPLAIN (ANALYZE, BUFFERS)
SELECT created_at, total_cents FROM orders WHERE customer_id = 42;
```

You want to see **`Index Only Scan`**. Check the `Heap Fetches:` line. If it's high, run `VACUUM orders;` and re-run — heap fetches should drop toward zero as the visibility map is refreshed.

## Expected result

- Part A: right column order removes the `Sort` node; wrong order forces a sort or a worse scan.
- Part B: the partial index is a fraction of the full index's size and is used only for the `pending` predicate.
- Part C: `Index Only Scan` with `Heap Fetches` near 0 after `VACUUM`.

## Done when…

- [ ] `notes.md` shows both column-order plans (Part A) and names which avoided the sort.
- [ ] You recorded the size gap between the full and partial index and showed the partial one is used only for its predicate (Part B).
- [ ] You captured an `Index Only Scan` and the `Heap Fetches` count before/after `VACUUM` (Part C).

## Stretch

- Rebuild Part C's index *without* `INCLUDE`, putting `total_cents` in the key instead. Does it still give an index-only scan? What does it cost that `INCLUDE` doesn't?
- Add a second `INCLUDE` column (`status`) and a query that selects it. Confirm still index-only.
- Run `SELECT idx_scan FROM pg_stat_user_indexes WHERE indexrelname='ix_full_status';` after the exercise. Is that full status index earning its keep, or is it a drop candidate?

## Submission

Commit `notes.md` to your portfolio under `c33-week-06/exercise-03/`.
