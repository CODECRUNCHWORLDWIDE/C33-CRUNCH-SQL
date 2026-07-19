# Mini-Project — Cut a Slow Query with the Right Index

> Take a slow, realistic query on the `shop` database and drive it fast with the right index — then document the whole thing as a one-page performance writeup with **before/after `EXPLAIN` evidence**. This is the deliverable that proves you learned the week.

**Estimated time:** 3 hours, mostly Saturday.

This mini-project mirrors the real workflow of a database engineer handed a "the app is slow" ticket: reproduce it, measure it, form a hypothesis, apply the smallest fix, and prove the result with numbers a skeptical reviewer would accept. The artifact is a short **performance report** — the kind you'd attach to a pull request that adds an index.

---

## Deliverable

A directory in your portfolio `c33-week-06/mini-project/` containing:

1. `report.md` — the one-page writeup (structure below).
2. `queries.sql` — every query and `CREATE INDEX` you ran, in order, runnable top to bottom.
3. `before.txt` and `after.txt` — the raw `EXPLAIN (ANALYZE, BUFFERS)` output for the target query, before and after.

---

## The target query

Use this query against the seed `shop` database (built in Lecture 1). It's a plausible "customer activity feed" query and is slow with no supporting index:

```sql
SELECT o.id, o.status, o.total_cents, o.created_at
FROM orders o
WHERE o.customer_id = 4242
  AND o.status IN ('paid','shipped','delivered')
  AND o.created_at >= '2023-01-01'
ORDER BY o.created_at DESC
LIMIT 50;
```

You may **also** pick a second query of your own from the `shop` schema if you want a richer report — but the one above is required.

---

## Milestones

### Milestone 1 — Reproduce and measure (30 min)

- Start from a clean index state (drop non-PK indexes you added earlier).
- Run the target query under `EXPLAIN (ANALYZE, BUFFERS)`. Save to `before.txt`.
- Read the plan bottom-up. Identify: the scan node, the row estimate vs actual, the buffer count, whether there's a `Sort` node, and where the time goes.

### Milestone 2 — Hypothesize (30 min)

In `report.md`, write your hypothesis *before* building anything:

- Which predicate is selective? (`customer_id = 4242` is very selective; the status set and date range less so.)
- What column order should the index have? (Equality before range; match the `ORDER BY`.)
- Could it be partial or covering?

### Milestone 3 — Build and prove (45 min)

- Create your index (or indexes — but justify more than one).
- `ANALYZE orders;`
- Re-run the query under `EXPLAIN (ANALYZE, BUFFERS)`. Save to `after.txt`.
- Compute: speedup (before ÷ after execution time) and buffer reduction. Confirm the scan node changed and any `Sort` disappeared.

### Milestone 4 — Interrogate the cost (30 min)

- Measure your index's size (`pg_relation_size`).
- Estimate/observe its write cost: time an insert of 50k orders with and without the index present.
- State honestly: is this index worth it for this workload? Under what write:read ratio would you *not* add it?

### Milestone 5 — Write the report (45 min)

Assemble `report.md` (see structure). Read it out loud once; tighten.

---

## `report.md` structure

```
# Performance report: customer activity feed query

## Problem
One paragraph: what's slow, on what data size.

## Baseline
The before-plan (or a link to before.txt). Call out the scan node,
the estimate-vs-actual, the buffers, and the total time.

## Hypothesis
What you predicted and why (selectivity, column order, covering).

## Fix
The exact index(es) created, with rationale for column order and any
partial/INCLUDE choices.

## Result
Before vs after table (time, buffers, scan node). The speedup number.
Link to after.txt.

## Cost & recommendation
Index size, write-cost observation, and your recommendation:
ship it / don't / ship a leaner variant. State the trade-off explicitly.
```

---

## Acceptance criteria

- [ ] `before.txt` and `after.txt` are real `EXPLAIN (ANALYZE, BUFFERS)` output for the required target query.
- [ ] The after-plan shows a different, better scan node (`Index Scan` / `Index Only Scan`) and the `LIMIT` is satisfied without scanning millions of rows.
- [ ] `report.md` has all six sections, with a computed speedup and buffer reduction.
- [ ] You state the index's **write-side cost** and give a clear ship/don't-ship recommendation — not just "it's faster."
- [ ] `queries.sql` runs top-to-bottom on a fresh `shop` database and reproduces your result.

---

## Rubric

| Criterion | Weight | "Great" looks like |
|-----------|------:|--------------------|
| Measurement rigor | 25% | Before/after with buffers, not just wall-clock; warm-cache caveat noted |
| Index design | 25% | Correct column order; partial/covering used (or rejected) with reasoning |
| Speedup achieved | 20% | Large, real improvement; `LIMIT` short-circuits via the index |
| Cost analysis | 15% | Index size + write cost measured, honest recommendation |
| Report clarity | 15% | A reviewer could approve the PR from your report alone |

---

## Why this matters

This is, in miniature, the exact deliverable Week 7 (query optimization) and Week 12 (the capstone: tune a database to a latency target) scale up. "It's faster, trust me" gets rejected in code review; "here's the before/after plan, here's the buffer drop, here's what it costs on writes" gets merged. You're building the habit now, on a query small enough to fit on one page.

---

When done: push, then start [Week 7 — Query optimization & plans](../../week-07-query-optimization-and-plans/).
