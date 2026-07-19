# Week 12 — Homework

Six problems, ~5 hours total. They reinforce the week without repeating the capstone. Commit each to your portfolio under `c33-week-12/homework/`. Use the `crunch_tune` database from the exercises unless a problem says otherwise.

---

## Problem 1 — Read three plans cold (45 min)

Without running them first, predict the plan for each query, then check with `EXPLAIN (ANALYZE, BUFFERS)`:

```sql
-- a
SELECT count(*) FROM orders WHERE status = 'refunded';
-- b
SELECT * FROM orders WHERE id = 4000000;
-- c
SELECT customer_id, count(*) FROM orders GROUP BY customer_id ORDER BY count(*) DESC LIMIT 5;
```

**Deliver** `plans.md`: for each, your prediction (scan type, rough rows), the actual plan, and one sentence on where your prediction was wrong and why. Being wrong and understanding why is the point.

---

## Problem 2 — The total_exec_time hunt (45 min)

Reset `pg_stat_statements`, replay a mixed workload (the four exercise queries, different literals, many times), then answer in `hunt.md`:

1. Which query has the highest **total** time? The highest **mean**? Are they the same query?
2. Which query would you optimize first, and why is it *not necessarily* the one with the highest mean?
3. Show the `SELECT` you used against `pg_stat_statements`, including the `pct` of total column.

---

## Problem 3 — Prove an index can hurt (45 min)

On a copy of `orders`, measure the write cost of over-indexing:

```sql
CREATE TABLE orders_w AS SELECT * FROM orders LIMIT 500000;
-- time a bulk insert with NO extra indexes
-- then add 4 indexes and time the same insert again
```

**Deliver** `index-cost.md`: the insert time with 0 vs. 4 indexes, the total index size added, and a paragraph on when this write tax is worth paying and when it is not.

---

## Problem 4 — Fix a wrong type (45 min)

Add a deliberately wrong column and fix it:

```sql
ALTER TABLE orders ADD COLUMN amount_text text;
UPDATE orders SET amount_text = total::text;   -- pretend money was stored as text
```

Now: write a query that sums `amount_text` (it must cast every row), measure it, then migrate the column to `numeric` and measure again. **Deliver** `type-fix.md` with the `ALTER TABLE ... USING` migration, before/after timing, and the risks of running that migration on a live 5M-row table (lock, rewrite time, mitigation).

---

## Problem 5 — Spot the N+1 (30 min)

Here is application pseudo-code:

```
customers = query("SELECT id FROM customers WHERE country = 'US' LIMIT 100")
for c in customers:
    orders = query("SELECT * FROM orders WHERE customer_id = $1", c.id)
```

**Deliver** `n-plus-one.md`: (1) how many queries this runs; (2) how it would appear in `pg_stat_statements`; (3) the single set-based query that replaces it; (4) the before/after query count.

---

## Problem 6 — Capacity projection (45 min)

Using `pg_total_relation_size` and the row counts, project the `crunch_tune` database forward. **Deliver** `capacity.md`:

1. Current heap + index size of `orders`, and bytes/row.
2. At 200k new orders/day, GB/year for heap and for heap+indexes.
3. At what point (roughly) does the working set exceed a 16 GB RAM box, and what is your first move when it does (partition, archive, replica, bigger box)?
4. One sentence: which monitoring signal would warn you this is coming *before* users feel it?

---

## Time budget

| Problem | Time |
|--------:|----:|
| 1 | 45 min |
| 2 | 45 min |
| 3 | 45 min |
| 4 | 45 min |
| 5 | 30 min |
| 6 | 45 min |
| **Total** | **~4.25 h** |

After homework, ship the [capstone](./mini-project/README.md) and take the [quiz](./quiz.md).
