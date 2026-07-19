# Exercise 1 — Two-Table Joins

**Goal:** Get fluent with the everyday `INNER JOIN` between two tables — aliases, `ON`, `USING`, and predicting row counts. By the end, a two-table join is something you type without thinking.

**Estimated time:** 40 minutes.

## Setup

Load the `crunch_shop` seed database first — see [exercises/README.md](./README.md). Then open a shell:

```bash
psql crunch_shop          # PostgreSQL
# or
sqlite3 crunch_shop.db    # SQLite  (.headers on / .mode column for readable output)
```

Write each answer into `solutions.sql`. Run it, and paste the row count you got as a `-- N rows` comment beneath.

## Warm-up — see the product first

Before joining, look at what you're filtering *down from*:

```sql
SELECT count(*) FROM customers CROSS JOIN orders;   -- 7 × 7 = 49
```

That 49 is every possible pairing. Every query below is that product, filtered by an `ON` condition. Keep the number in mind — a correct inner join here returns far fewer than 49.

## The eight tasks

1. **Every order with its customer's name.** Join `orders` to `customers`. Return `order_id`, customer `name`, `order_date`. *(Expect 7 rows.)*

2. **The same query, written with `USING`.** Both tables name the key `customer_id`, so `JOIN customers USING (customer_id)` is legal. Confirm the row count is identical to task 1, and note that `customer_id` now appears as a single merged column.

3. **Every order line with the product name and unit price.** Join `orders` → `order_items` → `products`. Return `order_id`, product `name`, `oi.quantity`, `oi.unit_price`. *(Expect 10 rows — one per line item. Explain in a comment why it's 10 and not 7.)*

4. **Line-item revenue.** Extend task 3 with a computed column `oi.quantity * oi.unit_price AS line_total`. Which single line item has the largest `line_total`? *(Hint: `ORDER BY line_total DESC LIMIT 1`.)*

5. **Each order with the sales rep who took it.** Join `orders` to `employees` on `employee_id`. Return `order_id`, employee `name AS rep`, `status`.

6. **Products with their supplier's name.** Join `products` to `suppliers`. Return product `name`, supplier `name AS supplier`. *(Watch out: product 6 has a `NULL` supplier_id — with an `INNER JOIN` it drops. How many rows do you get? Explain.)*

7. **Orders placed by North-region customers.** Join `orders` to `customers`, then filter `WHERE c.region_id = 1`. Return `order_id`, customer `name`. *(Which customers qualify?)*

8. **Three-table join:** every order line showing customer, product, and quantity. Join `orders` → `customers` → `order_items` → `products`. Return `order_id`, customer `name`, product `name`, `quantity`, ordered by `order_id` then product name.

## Expected results (check yourself)

<details>
<summary>Row counts — reveal after attempting</summary>

- Task 1: **7** rows.
- Task 3: **10** rows — `order_items` has 10 rows; the join to `orders`/`products` doesn't reduce that because every item has a valid order and product.
- Task 4: the largest `line_total` is order **105**, product **Nut**: `100 × 1.20 = 120.00`.
- Task 6: **5** rows — product 6 (Gizmo-Pro) has `supplier_id = NULL`, so the inner join drops it. A `LEFT JOIN` (Exercise 2) would keep it.
- Task 7: **3** rows — orders 101, 102 (Acme) and 103 (Bolt Co), both North.
- Task 8: **10** rows.

</details>

## Done when…

- [ ] `solutions.sql` has all eight queries, each with its row-count comment.
- [ ] Every column in every join is qualified with a table alias (`c.name`, not bare `name`).
- [ ] You can explain, in one sentence, why task 3 returns 10 rows but task 1 returns 7.
- [ ] You can explain why task 6 returns 5, not 6.

## Stretch

- Rewrite task 8 to also show the **category name** (add a fourth/fifth join). What happens to Gizmo-Pro's line if you use `INNER JOIN categories`? (Preview of Exercise 2.)
- Add the sales rep's **manager name** to task 5 using a self-join on `employees` (preview of Exercise 3).

## Submission

Commit `solutions.sql` to your portfolio under `c33-week-02/exercise-01/`.
