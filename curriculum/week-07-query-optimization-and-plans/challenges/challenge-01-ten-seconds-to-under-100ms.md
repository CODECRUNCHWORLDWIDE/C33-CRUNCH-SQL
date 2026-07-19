# Challenge 1 — Ten Seconds to Under 100ms

**Scenario:** The analytics team ships a "monthly revenue by tier" query. On the production-sized dataset it takes about ten seconds, and the dashboard times out. Your job: drive it under **100 ms** — and prove it with plans.

**Estimated time:** 1.5 hours.

## The offending query

Load the Week 7 dataset ([`exercises/README.md`](../exercises/README.md)), then meet the enemy. It's written the way a hurried analyst writes SQL — correct results, terrible plan:

```sql
SELECT c.tier,
       EXTRACT(year FROM o.created_at)  AS yr,
       EXTRACT(month FROM o.created_at) AS mo,
       count(*)      AS orders,
       sum(o.total)  AS revenue
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'shipped'
  AND date_trunc('month', o.created_at) >= '2024-06-01'
GROUP BY c.tier, EXTRACT(year FROM o.created_at), EXTRACT(month FROM o.created_at)
ORDER BY yr, mo, c.tier;
```

## Your task

Get the same result set, faster than **100 ms** on a warm cache. There are several legitimate paths; you decide which to combine.

## Constraints

- The **result set must be identical** (same rows, same values). Verify with `EXCEPT` against the original if unsure.
- You may add indexes, rewrite the query, add statistics, and set session GUCs (`work_mem`, `random_page_cost`). You may **not** pre-aggregate into a summary table for this challenge (that's a valid production tactic, but here we're drilling query + index + stats tuning).
- Report the *warm* run (second execution).

## Where to look (hints, in escalating order)

<details>
<summary>Hint 1 — read the baseline plan first</summary>

Run `EXPLAIN (ANALYZE, BUFFERS)` on the original. Find the node with the worst estimated-vs-actual rows, and note whether the big scan is a Seq Scan filtering millions of rows. Do not change anything yet.

</details>

<details>
<summary>Hint 2 — the predicate is hiding the column</summary>

`date_trunc('month', o.created_at) >= '2024-06-01'` is non-sargable — the index on `created_at` can't be used. Rewrite it as a bare-column range: `o.created_at >= '2024-06-01'`. (Confirm the boundary semantics still match the original — `date_trunc('month', x) >= '2024-06-01'` is equivalent to `x >= '2024-06-01'` here since the 1st is a month start.)

</details>

<details>
<summary>Hint 3 — give the scan an index it can actually use</summary>

You're filtering on `status = 'shipped'` AND a `created_at` range. A composite index — and possibly a *covering* one that INCLUDEs `total` and `customer_id` — can turn the big Seq Scan into an Index Only or Bitmap scan. Think about column order (equality column first, range column second).

</details>

<details>
<summary>Hint 4 — the GROUP BY expressions</summary>

Grouping by `EXTRACT(...)` twice recomputes functions per row. It's usually cheap relative to the scan, but once the scan is fixed, check whether the sort/group node is now the bottleneck and whether a bigger `work_mem` keeps it in memory.

</details>

## How success is judged

| Criterion | Weight | What "great" looks like |
|-----------|------:|-------------------------|
| Hit the target | 35% | Warm `Execution Time` < 100 ms, shown in the after-plan |
| Correct results | 20% | `EXCEPT` both ways returns zero rows — identical output |
| Evidence | 20% | Before-plan and after-plan both pasted; the improved node is circled/annotated |
| Reasoning | 15% | You explain *why* each change worked, tied to the plan |
| Restraint | 10% | You used the fewest, least-invasive changes that hit the target |

## Deliverable

A write-up containing: the baseline plan + time, each change with a one-line justification, the final plan + time, and the `EXCEPT` verification. State your final speedup as `Xs → Yms`.
