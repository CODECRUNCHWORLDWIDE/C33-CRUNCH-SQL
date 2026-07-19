# Challenge 1 — Layered Analytics

**Time:** ~75 minutes. **Difficulty:** Medium. **No single right answer.**

## Scenario

The founder of the `crunch_shop` store wants a **one-query customer scorecard**: a single result set, one row per customer, that a non-technical person could read over coffee. It must include *every* customer — even ones who have never ordered — and it must not let refunded or cancelled orders inflate the revenue.

## What the result must contain

One row per customer (**all 8**, including Hana Kim who never ordered), with:

| Column | Definition |
|--------|------------|
| `full_name` | the customer |
| `country` | their country |
| `total_orders` | count of their orders in *any* status |
| `paid_orders` | count of their `paid` orders only |
| `net_revenue` | revenue from `paid` orders only (`quantity * unit_price`), **0 if none** |
| `last_order_date` | date of their most recent order in any status, or NULL if none |
| `segment` | `'whale'` if net_revenue ≥ 1000, `'regular'` if > 0, `'dormant'` if 0 |

Order the result by `net_revenue` descending, then `full_name`.

## Constraints

- **Every customer appears** — Hana Kim must be a row with zeros and a `dormant` segment, not missing. This forces a `LEFT JOIN` (or a correlated subquery per column); an inner join silently drops her.
- **Refunded/cancelled orders count toward `total_orders` but not `net_revenue`.** Getting this right is the crux — a plain `WHERE status='paid'` up front would wrongly zero out `total_orders` too.
- Build it from **named CTEs**, not one deeply nested subquery. A reader should be able to point at each stage.
- No hard-coded customer ids or counts anywhere.

## Hints

<details>
<summary>Structuring the stages</summary>

One clean shape: a CTE for order-level facts per customer (`total_orders`, `paid_orders`, `last_order_date`), a second CTE for `net_revenue` per customer (from paid order lines only), then `customers LEFT JOIN`ed to both, with `COALESCE(..., 0)` to turn the missing rows into zeros.

</details>

<details>
<summary>The "count all but sum only paid" trick</summary>

`COUNT(*)` counts every order; `COUNT(*) FILTER (WHERE status='paid')` counts only paid — in the *same* grouped query, no separate pass. That's exactly what `FILTER` is for. On old SQLite, use `SUM(CASE WHEN status='paid' THEN 1 ELSE 0 END)`.

</details>

<details>
<summary>Deriving the segment</summary>

A `CASE` expression over the final `net_revenue`. Compute `net_revenue` in a CTE first so the `CASE` can reference it by name instead of repeating the whole revenue expression.

</details>

## How success is judged

| Criterion | What "great" looks like |
|-----------|-------------------------|
| Completeness | all 8 customers present; Hana Kim is `dormant` with zeros and a NULL last-order date |
| Correctness | net_revenue counts *only* paid lines; total_orders counts *all* statuses; numbers match a hand count |
| Edge handling | no NULLs leaking into `net_revenue` (COALESCE'd to 0); refunds excluded from revenue but not from total_orders |
| Readability | named CTEs, one job each; a teammate could read it unaided |
| Segmentation | whale/regular/dormant thresholds applied to the correct figure |

## Verify yourself

Hand-check at least three rows against the raw data before declaring victory. For instance: Ana Ruiz (customer 1) has 2 orders, both paid, net revenue 1509.00 → `whale`. Elif Demir (customer 5) has 2 orders (one cancelled, one paid), net revenue 80.00 → `regular`. Hana Kim → 0 orders, 0 revenue, `dormant`.

## Submission

Commit `challenge-01.sql` plus a 4–6 line note: which CTE structure you chose, and one alternative (e.g. all-correlated-subqueries) you rejected and why.
