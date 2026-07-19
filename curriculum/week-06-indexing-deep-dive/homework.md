# Week 6 — Homework

Six problems, ~4.5 hours total. All against the `shop` seed database. Commit each answer to your portfolio under `c33-week-06/homework/`. As always this week: **no claim without a plan.**

---

## Problem 1 — Read three plans cold (30 min)

Run each query under `EXPLAIN (ANALYZE, BUFFERS)` on a clean index state (only PKs) and, for each, write two sentences: what scan node the planner chose, and why.

```sql
SELECT * FROM orders WHERE id = 999999;
SELECT * FROM orders WHERE status = 'delivered';
SELECT * FROM orders WHERE created_at = '2023-06-15 12:00:00+00';
```

**Acceptance:** `problem1.md` with the three plans and your two-sentence explanation each. Bonus: for query 2, state the approximate selectivity and why a `Seq Scan` is correct.

---

## Problem 2 — Sargability rewrite drill (45 min)

Each predicate below is non-sargable. Rewrite it to be sargable (or state that it needs an expression/opclass index and write that index). Prove the rewrite uses the index with before/after plans for at least three of them.

```sql
WHERE date(created_at) = '2023-06-15'
WHERE total_cents / 100 >= 500
WHERE lower(email) = 'user9@example.com'
WHERE created_at + interval '30 days' < now()
WHERE email LIKE 'anna%'          -- careful: what does the default collation do here?
```

**Acceptance:** `problem2.md` with each rewrite (or index), and three before/after plans.

---

## Problem 3 — Pick the index type (45 min)

For each requirement, name the index **type** (btree/hash/GIN/GiST/BRIN) and write the `CREATE INDEX`. Then build and test at least three.

1. Containment query `payload @> '{"action":"purchase"}'` on `events`.
2. Range scan on `orders.created_at` where the table is append-ordered and you want the **smallest possible** index.
3. Full-text search over `customers.bio`.
4. Overlap query on a `tstzrange` "availability" column (invent a small table).
5. `LIKE '%example%'` (unanchored) on `customers.email`.

**Acceptance:** `problem3.md` mapping each requirement → index type + DDL, with a one-line justification, and at least three tested with plans.

---

## Problem 4 — Composite column order (30 min)

Given this query, design the best single composite index and prove it removes the `Sort` node:

```sql
SELECT * FROM orders
WHERE customer_id = 77 AND status = 'delivered'
ORDER BY created_at DESC
LIMIT 25;
```

Then explain in two sentences why the order you chose beats the reverse order.

**Acceptance:** `problem4.md` with the DDL, before/after plans, and the two-sentence justification.

---

## Problem 5 — Make it index-only (30 min)

Turn this query into an `Index Only Scan`:

```sql
SELECT customer_id, created_at, total_cents
FROM orders
WHERE customer_id BETWEEN 100 AND 200;
```

Build the covering index, run `VACUUM orders;`, and show `Heap Fetches` dropping toward zero.

**Acceptance:** `problem5.md` with the covering-index DDL and before/after plans showing the `Index Only Scan` and the `Heap Fetches` count pre/post `VACUUM`.

---

## Problem 6 — Find and justify a dead index (30 min)

Add three plausible-looking but pointless indexes to `orders` (e.g. an index on a low-selectivity column no query filters by). Run a handful of realistic queries. Then use the catalog to find which indexes were never used:

```sql
SELECT indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE relname = 'orders'
ORDER BY idx_scan ASC, pg_relation_size(indexrelid) DESC;
```

Decide which to `DROP INDEX CONCURRENTLY` and justify keeping the rest.

**Acceptance:** `problem6.md` with the catalog output, your drop decisions, and a sentence each on why the survivors stay. Note why `CONCURRENTLY` matters in production.

---

## Time budget

| Problem | Time |
|--------:|----:|
| 1 | 30 min |
| 2 | 45 min |
| 3 | 45 min |
| 4 | 30 min |
| 5 | 30 min |
| 6 | 30 min |
| **Total** | **~3.5–4.5 h** |

After homework, ship the [mini-project](./mini-project/README.md) and take the [quiz](./quiz.md).
