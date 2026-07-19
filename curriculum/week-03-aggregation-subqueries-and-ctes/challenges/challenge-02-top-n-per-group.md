# Challenge 2 — Top-N per Group

**Time:** ~75 minutes. **Difficulty:** Medium-hard. **No single right answer.**

## Scenario

"Show me the **two best-selling products in each category**." It sounds trivial. It is one of the most instructive queries in all of SQL, because the obvious `GROUP BY` gives you the *best per category* (top-1) easily but the *top-2* only with real thought — and there are at least three genuinely different ways to do it. You will build it **without** window functions (those are Week 8, and doing it the hard way first is the point).

## The task

Rank products by **total revenue** (`SUM(quantity * unit_price)` across all their order lines, any status) *within each product's category*, and return the **top 2 per category**.

Return: `category_id`, `product_name`, `product_revenue`, and the product's rank within its category (1 or 2). Order by `category_id`, then rank.

*Expected (spot-checks):*

- Category 4 (Laptops): `Featherweight Laptop` 2598.00 (rank 1), `Workhorse Laptop` 1798.00 (rank 2).
- Category 5 (Accessories): `Noise-Cancel Headset` 398.00 (rank 1), `Mechanical Keyboard` 240.00 (rank 2) — `USB-C Hub` (135.00) is cut.
- A category with only one sold product returns just that one row.

## Constraints

- **No window functions** (`ROW_NUMBER`, `RANK`, …). Save those for Week 8; here you build the intuition they automate.
- A category with fewer than 2 sold products should return however many it has — don't fabricate rows.
- Ties are unlikely in this dataset, but state in a comment how your approach would behave if two products tied for 2nd.

## Three approaches (pick one, understand all three)

<details>
<summary>Approach A — correlated counting subquery</summary>

For each product, count how many products *in the same category* have **strictly greater** revenue. A product is in the top 2 if that count is `< 2` (0 means it's #1, 1 means it's #2). This is the canonical window-function-free technique:

```
rank_within_category = (number of same-category products with higher revenue) + 1
keep rows where rank_within_category <= 2
```

You'll want a CTE that computes `product_revenue` per product first, then a correlated subquery over that CTE.

</details>

<details>
<summary>Approach B — lateral / per-category subquery</summary>

For each category, select its products ordered by revenue and take the first 2. In PostgreSQL a `LATERAL` join with `ORDER BY revenue DESC LIMIT 2` does this directly. SQLite lacks `LATERAL`; you'd fall back to Approach A.

</details>

<details>
<summary>Approach C — self-join and count</summary>

Join the per-product-revenue set to itself on matching category and "other.revenue >= this.revenue", `GROUP BY` the product, and keep rows with `COUNT(*) <= 2`. Same idea as A expressed as a join. Watch the tie/`>=`-vs-`>` boundary carefully.

</details>

## Hints

<details>
<summary>Build the revenue base first</summary>

Every approach starts from the same CTE: `product_id, category_id, SUM(quantity*unit_price) AS product_revenue`, grouped per product, joined through `order_items`. Get that right and correct, then layer the ranking on top.

</details>

<details>
<summary>Off-by-one on the rank</summary>

"Count of products with *strictly greater* revenue" gives rank − 1. So rank 1 ⇒ zero products above it. Keep `count_above < 2`. If you use `>=` by mistake you'll count the product against itself — a classic bug. Test on category 4 where you know the answer.

</details>

## How success is judged

| Criterion | "Great" |
|-----------|---------|
| Correctness | top-2 per category matches a hand count; the spot-checks above pass |
| Edge handling | single-product categories return one row; unsold products contribute 0 or are excluded consistently |
| No window functions | ranking done with subquery/CTE/self-join, not `ROW_NUMBER` |
| Clarity | a `product_revenue` CTE, then a legible ranking step |
| Reflection | your comment on tie behaviour is accurate for the approach you chose |

## Stretch

- Generalise to **top-N**: make `2` a value you change in exactly one place.
- Redo it with a window function after you've read Week 8's lecture, and compare line count and readability. This is the "before/after" that sells window functions.
- Rank by category *including sub-categories* (roll a product up to its top-level category using the recursive category CTE from Exercise 3).

## Submission

Commit `challenge-02.sql` with the approach you chose, plus a comment naming one approach you rejected and the trade-off that decided it.
