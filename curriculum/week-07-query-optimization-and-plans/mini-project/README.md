# Mini-Project — Tune a 10-Second Query to Under 100ms

> Take a realistic, slow analytical query, profile it properly, and drive it below a hard latency target — then hand in the plan evidence that proves you did it. This is the week's capstone: everything from all three lectures, applied end to end.

**Estimated time:** 6–7 hours, spread across Thursday–Saturday.

This mini-project mirrors the real job. A stakeholder hands you a query that "is too slow." You don't guess — you profile, you read the plan, you form a hypothesis, you change one thing, you re-measure, and you keep a log the whole way. The deliverable is not just a fast query; it's a **tuning report** a colleague could learn from.

---

## The workload

Build a dataset large enough that tuning is real. Use the Week 7 exercise dataset ([`../exercises/README.md`](../exercises/README.md)) as the base, then add a `line_items` child table so joins have depth:

```sql
-- Run after the exercises/README.md dataset is loaded
DROP TABLE IF EXISTS line_items;
CREATE TABLE line_items (
  id         bigint PRIMARY KEY,
  order_id   bigint NOT NULL REFERENCES orders(id),
  sku        text NOT NULL,
  qty        int NOT NULL,
  unit_price numeric(10,2) NOT NULL
);

INSERT INTO line_items (id, order_id, sku, qty, unit_price)
SELECT g,
       1 + (g % 3000000),
       'SKU-' || (g % 5000),
       1 + (g % 5),
       round((random() * 100)::numeric, 2)
FROM generate_series(1, 9000000) g;

ANALYZE;   -- note: no index on line_items.order_id yet — that's part of the point
```

Nine million line items, deliberately under-indexed. Here is the target query — "top 20 SKUs by revenue for enterprise customers in H2 2024," written naively:

```sql
SELECT li.sku,
       sum(li.qty * li.unit_price) AS revenue,
       count(DISTINCT o.id)        AS orders
FROM line_items li
JOIN orders o    ON o.id = li.order_id
JOIN customers c ON c.id = o.customer_id
WHERE c.tier = 'enterprise'
  AND o.status = 'shipped'
  AND date_trunc('month', o.created_at) BETWEEN '2024-07-01' AND '2024-12-01'
GROUP BY li.sku
ORDER BY revenue DESC
LIMIT 20;
```

On the un-indexed dataset this runs in the multi-second range. **Your target: warm `Execution Time` under 100 ms, with identical results.**

---

## Deliverable

A directory in your portfolio at `c33-week-07/mini-project/` containing:

1. `baseline.md` — the original query, its `EXPLAIN (ANALYZE, BUFFERS)` plan, and warm timing.
2. `tuning-log.md` — a numbered log: each step = hypothesis, the one change you made, the new plan, the new time, and keep/revert decision.
3. `final.sql` — the final query text plus every DDL change (indexes, statistics) you applied.
4. `final.md` — the final `EXPLAIN (ANALYZE, BUFFERS)` plan, the warm timing, and the `EXCEPT`-both-ways proof that results are unchanged.
5. `report.md` — a one-page narrative: what was slow, why, what you changed, and the final `Xs → Yms` result.

---

## Suggested workflow

### Phase 1 — Profile (1–1.5h)

- Run the query under `EXPLAIN (ANALYZE, BUFFERS)`, twice, keep the warm run → `baseline.md`.
- Read it with the Lecture 1 routine: bottom-up, estimated vs actual, multiply by `loops`, check `Buffers`.
- Write down, in `baseline.md`, the **three** biggest problems you see before changing anything. (Likely: no index on `line_items.order_id` forcing a huge join, a non-sargable date predicate, and a bad row estimate somewhere.)

### Phase 2 — Fix one thing at a time (2–3h)

Work the Lecture 3 workflow. Typical order for this query:

1. **Make the date predicate sargable** — bare `o.created_at` range instead of `date_trunc(...)`.
2. **Index the join** — `line_items(order_id)` is almost certainly missing; without it the 9M-row join is brutal.
3. **Index the filter** — a composite/covering index on `orders(status, created_at)` including `customer_id`.
4. **Refresh stats** — `ANALYZE` after DDL; add `CREATE STATISTICS` if a multi-column estimate is off.
5. **Memory** — if a sort or hash spills (`Sort Method: external merge`), raise `work_mem` for the session.

After **each** change: re-run `EXPLAIN (ANALYZE, BUFFERS)`, log the new time, decide keep or revert. Never bundle changes.

### Phase 3 — Verify + write up (1.5–2h)

- Prove correctness: `(<original query>) EXCEPT (<final query>)` and the reverse both return zero rows.
- Capture the final plan and timing → `final.md`.
- Write `report.md`: the story, not just the numbers. Why was it slow? Which single change bought the most? What did you try that *didn't* help?

---

## Acceptance criteria

- [ ] Warm `Execution Time` in `final.md` is **under 100 ms**.
- [ ] `EXCEPT` both directions returns zero rows — results are provably identical.
- [ ] `tuning-log.md` shows at least **four** discrete steps, each with a before/after time and a keep/revert call.
- [ ] Every index or statistics object you created is in `final.sql` and justified in the log.
- [ ] You changed exactly one thing per logged step (no bundled changes).
- [ ] `report.md` states the speedup as `Xs → Yms` and names the single most impactful change.

---

## Rubric

| Criterion | Weight | "Great" looks like |
|-----------|------:|--------------------|
| Hit the target | 25% | Under 100 ms warm, proven in the final plan |
| Correctness | 15% | `EXCEPT` proof included; output identical |
| Diagnostic quality | 25% | You identified the real bottleneck from the plan, not by trial and error |
| Methodology | 20% | One change per step, each re-measured, reverts recorded |
| Communication | 15% | `report.md` teaches — a peer could follow your reasoning |

---

## Stretch goals

- Get it under **50 ms**. What's the new bottleneck once the obvious ones are gone?
- Compare a covering (`INCLUDE`) index against a plain composite index on the same columns. Measure the `Heap Fetches` difference.
- Re-run the whole thing in **SQLite** (adapt the DDL; use `EXPLAIN QUERY PLAN` and `.timer on`). Which fixes transfer and which don't? Write a paragraph on why SQLite's nested-loop-only join model changes your index strategy.
- Turn the final query into a materialized view and measure the read latency. When is that the right call versus tuning the live query?

---

## Why this matters

This is Week 7's rehearsal for the Week 12 capstone, where you tune a whole database to a latency target under load. The muscle you build here — profile, hypothesize, change one thing, re-measure, prove it — is the exact loop you'll run on real production incidents for the rest of your career. The engineers people trust with the database are the ones who can show the two plans.

---

When done: push, then start [Week 8 — Window Functions & Advanced SQL](../../week-08-window-functions-and-advanced-sql/).
