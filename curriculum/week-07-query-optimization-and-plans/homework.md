# Week 7 — Homework

Six practice problems (~4 hours total). They reinforce the lectures and prepare you for the challenges and mini-project. Use the Week 7 dataset from [`exercises/README.md`](./exercises/README.md). Keep answers in a `homework.md` in your portfolio; paste the plan for anything you claim.

---

## 1. Narrate a plan (30 min)

Run the following and write a **paragraph** narrating it top-to-bottom as if explaining to a teammate — which node runs first, what feeds what, where the time goes.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.country, count(*), sum(o.total)
FROM orders o JOIN customers c ON c.id = o.customer_id
WHERE o.created_at >= '2025-01-01'
GROUP BY c.country
ORDER BY sum(o.total) DESC;
```

Your paragraph must correctly use the words: node, build side, estimated vs actual, and either "in memory" or "spilled."

---

## 2. Estimate before you run (30 min)

Before running each query below, **write down your prediction**: scan type(s) and join algorithm. Then run `EXPLAIN` and score yourself.

```sql
-- (a)
SELECT * FROM orders WHERE id = 999999;
-- (b)
SELECT * FROM orders WHERE status = 'shipped';
-- (c)
SELECT * FROM orders o JOIN customers c ON c.id = o.customer_id WHERE c.id = 7;
-- (d)
SELECT * FROM orders o JOIN customers c ON c.id = o.customer_id;
```

Report your prediction, the actual plan, and one sentence on any miss. Getting `(b)` and `(d)` right requires reasoning about *selectivity*, not habit.

---

## 3. Make a slow query sargable (40 min)

This query filters on a computed expression. Show its plan, rewrite it to be sargable, add any index needed, and show the improved plan and timing.

```sql
SELECT count(*) FROM orders
WHERE EXTRACT(year FROM created_at) = 2024 AND status = 'pending';
```

Deliver: before-plan, the rewritten query, the DDL you added (if any), after-plan, and the `Xs → Yms`.

---

## 4. `NOT IN` vs `NOT EXISTS` (30 min)

Create a small exclusion set and compare:

```sql
CREATE TABLE flagged (customer_id int);
INSERT INTO flagged VALUES (1),(2),(3),(NULL);  -- note the NULL

-- (a) NOT IN
SELECT count(*) FROM orders WHERE customer_id NOT IN (SELECT customer_id FROM flagged);
-- (b) NOT EXISTS
SELECT count(*) FROM orders o WHERE NOT EXISTS (SELECT 1 FROM flagged f WHERE f.customer_id = o.customer_id);
```

- Report the **result count** of each. Explain the difference (hint: the NULL).
- Report the plan of each. Which used an Anti Join?
- One sentence: why is `NOT EXISTS` the safer default?

---

## 5. Force a disk spill, then fix it (40 min)

Find a query that sorts a large set and make it spill, then keep it in memory:

```sql
SET work_mem = '64kB';   -- tiny, to force a spill
EXPLAIN (ANALYZE) SELECT * FROM orders ORDER BY total;
-- capture the 'Sort Method' line

SET work_mem = '256MB';
EXPLAIN (ANALYZE) SELECT * FROM orders ORDER BY total;
-- capture the new 'Sort Method' line
RESET work_mem;
```

Report both `Sort Method` lines and both timings. Write two sentences on what `work_mem` controls and why setting it high *globally* is risky.

---

## 6. Cost-constant experiment (30 min)

Pick the medium-selectivity query `SELECT * FROM orders WHERE status = 'pending';`. Show its plan at the default `random_page_cost`, then at `1.1`:

```sql
SHOW random_page_cost;                    -- default 4
EXPLAIN (ANALYZE) SELECT * FROM orders WHERE status = 'pending';
SET random_page_cost = 1.1;
EXPLAIN (ANALYZE) SELECT * FROM orders WHERE status = 'pending';
RESET random_page_cost;
```

Did the scan choice change? Explain in two sentences what `random_page_cost` models and why lowering it on SSD hardware is often correct.

---

## Submission

Commit `homework.md` (with all six answers and pasted plans) under `c33-week-07/homework/`. Total effort should be about four hours; if you're flying through, you're probably not reading the plans closely enough — slow down and narrate them.
