# Mini-Project — The Store Analytics Report (built from CTEs)

> Build **one** layered SQL query that turns the `crunch_shop` order data into a founder-ready analytics report — assembled from a chain of named CTEs, each doing one job. This is the week's capstone: it exercises aggregation, `HAVING`, a subquery, and a multi-stage `WITH` pipeline in a single, readable statement.

**Estimated time:** 4–5 hours, spread across the back half of the week.

## The goal

A retailer wants a monthly-ish snapshot they can paste into a board update. You will deliver a single query (plus a short write-up) that produces a **country performance report** with the metrics below — built as a pipeline of CTEs so that each stage is inspectable and the whole thing reads top to bottom.

You are not shipping ten separate queries. You are shipping **one** query whose stages are named, plus the reasoning that got you there.

## Deliverable

A directory in your portfolio `c33-week-03/mini-project/` containing:

1. `report.sql` — the single layered query (CTE chain). Runs top to bottom on the seed DB.
2. `report-output.txt` — the actual result, pasted from `psql`/`sqlite3`.
3. `writeup.md` — 250–400 words: what each CTE stage does, one edge case you had to handle, and one thing you'd add with next week's tools.

## What the report must contain

One row per **country**, only for countries with **at least one paid order**, containing:

| Column | Definition |
|--------|------------|
| `country` | the country |
| `customers` | distinct customers from that country who have ordered |
| `paid_orders` | number of paid orders |
| `net_revenue` | total revenue from paid order lines (`quantity * unit_price`) |
| `avg_order_value` | `net_revenue / paid_orders`, rounded to 2 decimals |
| `revenue_share` | this country's `net_revenue` as a % of **all** countries' net revenue, rounded to 1 decimal |
| `rank` | 1 = highest net_revenue (compute without a window function — a subquery/count is fine) |

Order by `net_revenue` descending.

*Spot-check:* US net revenue is 1509.00 and is the largest, so US is rank 1; the three qualifying countries' revenue_share values should sum to ~100.0%.

## Required structure

Build it as **at least three** named CTEs feeding a final `SELECT`. A clean decomposition:

1. `paid_lines` — paid order lines with a computed `line_total`.
2. `by_country` — group `paid_lines` to get `customers`, `paid_orders`, `net_revenue` per country. Apply the "at least one paid order" rule here (it's automatic once you group paid lines).
3. `totals` — a one-row CTE with the grand total net revenue (for `revenue_share`).
4. Final `SELECT` — join `by_country` to `totals`, compute `avg_order_value`, `revenue_share`, and `rank` (rank via a correlated count over `by_country`), order by `net_revenue`.

You may add stages or restructure — but every stage must be **named for what it holds** and do **one** job.

## Milestones

### Milestone 1 — the base (1h)
Write and verify `paid_lines` alone. Confirm it returns only paid lines with correct `line_total`s. Hand-check two rows.

### Milestone 2 — per-country aggregates (1h)
Add `by_country`. Verify `net_revenue` per country against your Exercise 1 answers (US 1509, CA 545, DE 324).

### Milestone 3 — shares and rank (1.5h)
Add `totals`, then compute `revenue_share` and `rank` in the final `SELECT`. Confirm the shares sum to ~100% and US ranks 1.

### Milestone 4 — polish + write-up (1h)
Format the query, align it, add brief comments, capture the output, and write `writeup.md`.

## Rubric

| Criterion | Weight | "Great" looks like |
|-----------|------:|--------------------|
| Correctness | 30% | Every metric matches a hand count on the seed data |
| CTE structure | 25% | ≥3 named, single-purpose stages; reads top to bottom |
| Edge handling | 15% | Only paid revenue counts; only paid-order countries appear; no divide-by-zero or NULL leaks |
| Derived metrics | 15% | `avg_order_value`, `revenue_share`, and `rank` computed correctly (rank without a window fn) |
| Write-up | 15% | Explains each stage clearly; names a real edge case and a next-week improvement |

## Why this matters

This is the shape of real analytics work: a business question decomposed into named, verifiable stages. The founder doesn't care whether you used a subquery or a CTE — they care that the number is right and that the next engineer can maintain the query. Decomposition-into-named-stages is *the* transferable skill of this week, and it's exactly what Week 8 (window functions) and the C27 Data course build on.

When done: push, then take the Week 3 [quiz](../quiz.md) and start [Week 4 — Data modeling & normalization](../../week-04-data-modeling-and-normalization/).
