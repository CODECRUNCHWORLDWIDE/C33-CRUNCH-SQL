# Exercise 1 — Group and Aggregate

**Goal:** Get fluent with the aggregate functions, `GROUP BY`, `HAVING`, and `FILTER` by answering eight business questions against `crunch_shop`.

**Estimated time:** 45 minutes.

## Before you start

Load the seed database (see `exercises/README.md`). Confirm the sanity check passes (8 customers, 9 products, 10 orders, 15 items). Write every answer into a `solutions.sql`; run each and eyeball the result against the expected value given.

Throughout, treat **line revenue** as `quantity * unit_price`.

## Tasks

### 1. Order counts by status

Return each `status` with the number of orders that have it, most common first.

*Expected:* `paid` 6, `shipped` 2, `refunded` 1, `cancelled` 1.

### 2. Revenue per country (paid only)

Join `orders → customers → order_items`, keep only `status = 'paid'`, and return each `country` with its total paid revenue, highest first.

*Expected:* `US` 1509.00, `CA` 545.00, `DE` 324.00. (`SE` has no paid orders, so it does not appear.)

Ask yourself: **why does `SE` disappear** rather than showing `0`? (Because there are no paid rows for it to group — an inner join drops it. Hold that thought for challenge 1.)

### 3. Big-revenue countries with HAVING

Extend task 2: keep only countries whose total paid revenue is **over 500**.

*Expected:* `US` 1509.00, `CA` 545.00.

Confirm you put `status = 'paid'` in `WHERE` and the revenue threshold in `HAVING` — and that you can say in one sentence why each belongs where it is.

### 4. Average price per category

Return each `category_id` with the number of products in it and their average price, rounded to 2 decimals.

*Hint:* `ROUND(AVG(price), 2)`.

### 5. FILTER — status breakdown in one row per country

For each `country`, produce a single row with: total orders, paid orders, refunded orders, and cancelled orders — using `COUNT(*) FILTER (WHERE …)`.

*Expected (partial):* `US` → 3 total, 2 paid, 0 refunded, 0 cancelled. `DE` → 3 total, 1 paid, 0 refunded, 1 cancelled.

If your SQLite build predates 3.30, rewrite the `FILTER` columns using `SUM(CASE WHEN … THEN 1 ELSE 0 END)` and confirm you get the same numbers.

### 6. Distinct products per customer

For each `customer_id` that has ordered, return the number of order lines and the number of **distinct** products purchased. Explain (in a comment) any customer where the two numbers differ.

### 7. Customers with more than one order

Return each `customer_id` who has placed **two or more** orders, with their order count.

*Expected:* customers 1, 3, and 5 — each with 2.

### 8. (Postgres only) Rollup subtotals

Using `GROUP BY ROLLUP (country, status)`, produce paid-plus-all revenue with country subtotals and a grand total. Use `GROUPING()` (or a `CASE`) to label the subtotal rows `ALL STATUSES` / `ALL COUNTRIES` instead of leaving NULLs.

If you're on SQLite, note in a comment that `ROLLUP` is unavailable and sketch how you'd emulate it with `UNION ALL`.

## Done when…

- [ ] `solutions.sql` contains all eight queries, each runnable top to bottom.
- [ ] Tasks 1, 2, 3, 5, and 7 match the expected values above.
- [ ] You can state, in one sentence each, the difference between `WHERE` and `HAVING` and between `COUNT(*)` and `COUNT(col)`.
- [ ] Your task-5 answer uses `FILTER` (or the documented `CASE` fallback) — not eight separate queries.

## Stretch

- Add a task-2 variant that includes **all** statuses and shows revenue split by status per country in one result.
- Compute the **overall** average order value (revenue per order) as a single number, then list orders above it (you'll want a subquery — a preview of Exercise 2).
- Rewrite task 8's rollup as three explicit `GROUPING SETS` and confirm the output is identical.
