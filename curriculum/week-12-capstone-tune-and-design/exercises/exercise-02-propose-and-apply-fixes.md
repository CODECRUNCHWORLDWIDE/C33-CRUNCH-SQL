# Exercise 2 — Propose and apply fixes

**Goal:** For each of the three offenders you found in Exercise 1, form a written hypothesis, then apply **exactly one** fix and observe the plan change. One lever per step — so you always know what caused what.

**Estimated time:** ~2 hours.

**Prerequisite:** Exercise 1 complete; `crunch_tune` seeded and profiled; `report.md` lists the three worst queries.

## The rule for this exercise

Change one thing. Re-read the plan. Write down what happened. Only then move to the next change. If you batch three `CREATE INDEX`es and re-run, you will not know which one mattered — and in the capstone report, "I added three indexes and it got faster" is not an answer.

## Step 1 — Fix Q2 (customer order history)

**Hypothesis (write it first):** "Q2 does a `Seq Scan` over 5M rows to find one customer's ~25 orders because `customer_id` is unindexed. A B-tree on `customer_id` should flip it to an `Index Scan` and cut buffers by ~1000×."

Apply one lever:

```sql
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders (customer_id);
```

Re-check the plan:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, status, total, created_at
FROM orders WHERE customer_id = 12345 ORDER BY created_at DESC;
```

Note in `report.md`: did the node change from `Seq Scan` to `Index Scan` (or `Bitmap Heap Scan`)? Did `Buffers: read` collapse? Did the estimate now match reality?

**Refinement (a second, separate step):** Q2 also `ORDER BY created_at DESC`. A composite index can serve *both* the filter and the sort, removing the `Sort` node:

```sql
CREATE INDEX CONCURRENTLY idx_orders_cust_created ON orders (customer_id, created_at DESC);
```

If this makes `idx_orders_customer` redundant (it is a prefix), decide whether to drop the single-column index — and justify it.

## Step 2 — Fix Q4 (refund count for a customer)

**Hypothesis:** "Q4 filters on `(customer_id, status)`. A composite index matching that predicate turns the count into an index scan over a few rows." Apply:

```sql
CREATE INDEX CONCURRENTLY idx_orders_cust_status ON orders (customer_id, status);
```

Re-plan Q4. Consider: is this index redundant with `idx_orders_cust_created`? (A `(customer_id, created_at)` index still helps Q4's `customer_id` lookup but cannot filter `status` in the index.) Decide which indexes to keep and write your reasoning — this is exactly the over-indexing trade from Lecture 2.

## Step 3 — Fix Q1 (recent high-value orders)

**Hypothesis:** "Q1 filters `created_at >= now() - 7 days` then sorts by `total DESC`. An index on `created_at` finds recent rows; add `total` so the range and the order both use the index." Try:

```sql
CREATE INDEX CONCURRENTLY idx_orders_created ON orders (created_at);
```

Re-plan. Then try a version that also covers the sort and compare which the planner prefers:

```sql
CREATE INDEX CONCURRENTLY idx_orders_created_total ON orders (created_at, total DESC);
```

Note which index the planner actually chose (`EXPLAIN` names it) — it may prefer the narrower one. The planner is the judge, not your intuition.

## Step 4 — Decide on Q3 (revenue by country)

Q3 aggregates the whole `paid` table — an index may **not** help, because you genuinely touch most rows. This is the "when NOT to add an index" lesson. Options to weigh:

- A partial index `WHERE status = 'paid'` — helps only if `paid` is a small fraction (here it is ~50%, so probably not).
- Leave the seq scan; a full aggregate over 5M rows may already be acceptable against its budget.
- If this report is run constantly, a **materialized view** refreshed hourly beats any index.

Write your decision and *why* — including the case for doing nothing.

## Expected result

- Q2 and Q4: `Seq Scan` → `Index Scan` / `Bitmap Heap Scan`, `Buffers: read` drops from tens of thousands to single/low double digits, warm time from ~hundreds of ms to under a millisecond.
- Q1: index chosen, sort removed or reduced, big drop in rows scanned.
- Q3: a *justified* decision (index, materialized view, or leave it) — not a reflex index.

## Done when…

- [ ] Each fix was applied and measured **one at a time**, with the plan re-read after each.
- [ ] `report.md` has, per query: hypothesis → the one change → the observed plan change.
- [ ] You made an explicit keep/drop decision about any redundant indexes and justified it.
- [ ] Q3 has a written decision, including the option of no index.
- [ ] Every index was built with `CREATE INDEX CONCURRENTLY`.

## Stretch

- Run `SELECT indexrelname, idx_scan FROM pg_stat_user_indexes WHERE relname='orders';` — which of your new indexes are actually being used?
- Deliberately add a useless index (e.g. on `country` where no query filters it) and confirm `idx_scan` stays 0 — that is dead weight you would remove in a review.

## Submission

Commit the updated `report.md` under `c33-week-12/exercise-02/`.
