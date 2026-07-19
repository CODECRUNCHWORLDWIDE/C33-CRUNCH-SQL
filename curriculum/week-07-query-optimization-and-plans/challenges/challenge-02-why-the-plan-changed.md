# Challenge 2 — Why the Plan Changed

**Scenario:** A query that used a fast Index Scan last month now uses a Seq Scan and is slower — but *nobody changed the query or the indexes.* The only thing that changed is that the table grew. Your job is not to "fix" it but to **explain, with evidence, exactly why the planner switched** — and then decide whether the switch was correct.

**Estimated time:** 1 hour.

This is the harder skill. Anyone can add an index. A senior engineer can look at two plans and narrate the planner's reasoning: "at 5% selectivity the index scan wins; the table grew, this value is now 30% of rows, so the seq scan is genuinely cheaper — the planner is right, and the real fix is elsewhere."

## Reproduce the flip

Start from the Week 7 dataset ([`exercises/README.md`](../exercises/README.md)). We'll make a value go from rare to common and watch the plan cross over.

### 1. A rare value uses the index

```sql
-- Make 'refunded' rare: only ~300 rows out of 3,000,000
UPDATE orders SET status = 'refunded' WHERE id % 10000 = 0;
ANALYZE orders;

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'refunded';
```

Record the plan. It should use the index (Index Scan or Bitmap Index Scan) because so few rows match.

### 2. Grow the value until the plan flips

```sql
-- Make 'refunded' common: now a large fraction of the table
UPDATE orders SET status = 'refunded' WHERE id % 3 = 0;
ANALYZE orders;

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'refunded';
```

Record the new plan. It should now be a Seq Scan.

## Your task

Write an explanation that answers all of these, each backed by a number from the plans or from `pg_stats`:

1. **What changed in the statistics?** Query `pg_stats` for `status` before and after. What did `most_common_freqs` for `'refunded'` become? What's the estimated selectivity in each case?
2. **What is the crossover point?** Roughly what fraction of the table must match before a Seq Scan beats an Index Scan *on this hardware*? Estimate it and explain the trade-off (random page reads vs one sequential sweep).
3. **Was the planner right?** Compare the actual `Execution Time` of both plans. If you *force* the index scan on the grown data (`SET enable_seqscan = off`), is it faster or slower than the Seq Scan the planner chose? What does that prove?
4. **What's the real fix?** If this query must stay fast even when `'refunded'` is common, what would you actually do — partial index, different query, covering index, partitioning? Justify.

## Constraints

- Every claim needs a number: a cost, a time, a frequency, or a `Buffers` count.
- You must include the `SET enable_seqscan = off` experiment and interpret it correctly.
- Reset any GUCs you set.

## How success is judged

| Criterion | Weight | What "great" looks like |
|-----------|------:|-------------------------|
| Correct root cause | 30% | You tie the flip to the change in estimated selectivity, with `pg_stats` numbers |
| Crossover reasoning | 25% | You articulate the random-vs-sequential trade-off and estimate the crossover fraction |
| Planner-was-right test | 25% | You force the index scan and correctly interpret whether the planner chose well |
| Real-fix judgment | 20% | Your proposed durable fix fits the scenario and you justify it |

## Deliverable

A write-up with: both plans, the `pg_stats` before/after for `status`, the forced-index experiment, and a clear narrative of the planner's reasoning. Clean up afterward: restore the `status` column or rebuild the dataset so later work starts fresh.
