# Challenge 1 — Cut a slow query with the right index

**Time:** ~75 minutes. **Difficulty:** Medium. **No single right answer.**

## Scenario

You're on-call for the `shop` analytics dashboard. A product manager reports that the "**top customers by spend in a date window**" tile takes several seconds and sometimes times out. Here's the query behind the tile:

```sql
SELECT c.id, c.email, c.country,
       count(o.id)          AS order_count,
       sum(o.total_cents)   AS total_spend
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'delivered'
  AND o.created_at >= '2023-06-01'
  AND o.created_at <  '2023-07-01'
GROUP BY c.id, c.email, c.country
ORDER BY total_spend DESC
LIMIT 20;
```

The database is the seed `shop` (2M orders, 100k customers). Assume no indexes beyond the two primary keys and the FK.

## Your task

Drive this query's execution time down as far as you reasonably can by adding the right index or indexes — then prove it. You decide what to build; there are several defensible designs.

## Constraints

- **You must start from a clean baseline.** Drop any non-PK indexes on `orders`/`customers` and capture the "before" plan with `EXPLAIN (ANALYZE, BUFFERS)`.
- **No more than three new indexes total.** Fewer is better; a one-index solution that wins is stronger than a three-index one.
- **No changing the query's results.** You may not add a materialized view or pre-aggregate table for this challenge — the point is indexing. (You *may* rewrite the query if it stays logically equivalent and returns the same rows.)
- **`created_at` is append-ordered** in the seed data — that's a hint, not a requirement to use.

## Hints

<details>
<summary>Where is the time going?</summary>

Read the baseline plan bottom-up. Is the expensive node the scan on `orders` (finding the delivered June orders), the join to `customers`, or the sort for `ORDER BY total_spend`? Fix the biggest cost first.
</details>

<details>
<summary>Which columns, in which order?</summary>

The filter on `orders` is an **equality** (`status = 'delivered'`) plus a **range** (`created_at` in June). Recall the rule: equality columns before range columns in a composite index. What composite index does that suggest? Would a **partial** index on `status = 'delivered'` be even leaner?
</details>

<details>
<summary>Can you make it index-only?</summary>

The aggregate needs `customer_id`, `total_cents`, and (for filtering) `status`, `created_at` from `orders`. If all of those live in one index, the `orders` side can be an `Index Only Scan` — no heap fetches for millions of rows. Think `INCLUDE`.
</details>

<details>
<summary>The join side</summary>

`customers.id` is already the PK, so the join lookup is indexed. Don't over-index the small side; the win is almost entirely on `orders`.
</details>

## How success is judged

| Criterion | What "great" looks like |
|-----------|-------------------------|
| Baseline captured | Before-plan pasted with buffers + time, indexes confirmed absent |
| Speedup | A large, measured drop in execution time **and** buffers (aim for 10×+; index-only can do far more) |
| Design justification | You explain column order, why partial/covering (or not), and the scan node the planner ended up using |
| Restraint | Solved with the fewest indexes; you note any index you *considered and rejected* and why |
| Trade-off awareness | One sentence on what your index costs on writes to `orders` |

## Deliverable

`challenge-01.md` containing:

1. The baseline plan.
2. Each index you created, with a one-line rationale.
3. The final plan, with the speedup and buffer reduction computed.
4. A short paragraph: why this design, what you rejected, and the write-side cost.

## Submission

Commit to your portfolio under `c33-week-06/challenge-01/`.
