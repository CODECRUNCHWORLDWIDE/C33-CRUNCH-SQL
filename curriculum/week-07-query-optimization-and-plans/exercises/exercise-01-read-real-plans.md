# Exercise 1 — Read Real Plans

**Goal:** Run five queries under `EXPLAIN (ANALYZE, BUFFERS)` and write, in plain English, what each plan does and why. By the end, reading a plan tree is automatic.

**Estimated time:** 35 minutes.

## Setup

Complete the dataset build in [`exercises/README.md`](./README.md) first. Then, in `psql`, turn on timing so you also see wall-clock:

```sql
\timing on
```

## The five plans

For each query below: run it under `EXPLAIN (ANALYZE, BUFFERS)`, paste the plan into `solutions.md`, and answer the questions in **one or two sentences each**. Run each query twice and read the *second* (warm) run.

### Plan A — a point lookup

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 4242;
```

- Which scan type did it choose? Why that one?
- Compare estimated `rows` to actual `rows`. Good estimate?

### Plan B — a wide range

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE total > 50;
```

- Which scan type now? Why *not* the index on nothing-in-particular here?
- Look at `Rows Removed by Filter`. What does it tell you?

### Plan C — a medium-selectivity value

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'pending';
```

- Did you get a Bitmap Heap Scan? Explain why bitmap and not a plain Index Scan.
- What is the `Recheck Cond` line doing there?

### Plan D — a join

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, c.city
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE c.country = 'FR';
```

- Which join algorithm? Name the build side and the probe side.
- Follow the tree: which node runs first? Which runs last?

### Plan E — a sort

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE created_at >= '2024-06-01'
ORDER BY total DESC
LIMIT 100;
```

- Find the `Sort` (or `Top-N`) node. Did it stay in memory or spill to disk? Which line told you?
- What did `LIMIT 100` let the executor avoid doing?

## Acceptance criteria

- [ ] `solutions.md` has all five plans pasted verbatim plus your written answers.
- [ ] For at least one plan you multiplied a per-node `actual time` by `loops` to get the real total.
- [ ] You correctly identified the scan type in all five and the join algorithm in Plan D.
- [ ] You can point to the exact line that proves whether Plan E's sort spilled to disk.

## Stretch

- Re-run Plan B with `SET random_page_cost = 1.1;` first. Did the scan choice change? Explain.
- Add `FORMAT JSON` to Plan D and paste it into <https://explain.dalibo.com/> for a visual tree. Does the visual match your reading?
- Run Plan A once cold (right after `DISCARD ALL;` won't flush disk cache, but the *first* run after connecting is coldest) and note the `read` vs `hit` difference in `Buffers`.

## Hints

<details>
<summary>Bitmap vs plain index scan (Plan C)</summary>

`status = 'pending'` matches ~20% of 3M rows = ~600k rows. That's too many for a plain Index Scan (600k random heap fetches) but selective enough that a full Seq Scan is wasteful on some hardware. The bitmap path reads matching *pages* once, in physical order — the middle-ground scan. Whether you actually get bitmap vs seq depends on `random_page_cost` and your row estimate.

</details>

## Submission

Commit `solutions.md` under `c33-week-07/exercises/exercise-01/`.
