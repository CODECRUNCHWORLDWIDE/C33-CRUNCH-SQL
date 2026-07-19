# Exercise 2 — Subqueries

**Goal:** Use all four subquery shapes — scalar, `IN`, `EXISTS`, and correlated — and feel the difference between an independent subquery and one that re-runs per outer row.

**Estimated time:** 50 minutes.

## Before you start

Seed database loaded and sanity-checked. Answers into `solutions.sql`.

## Tasks

### 1. Scalar subquery — above the average price

List products whose `price` is above the average price of all products. Return name and price, most expensive first.

*Hint:* `WHERE price > (SELECT AVG(price) FROM products)`.

### 2. IN — customers with a refunded order

List the `full_name` of every customer who has **at least one** refunded order, using `customer_id IN (subquery)`.

*Expected:* `Chen Wei` (customer 3 has the only refunded order).

### 3. NOT EXISTS — products never ordered

List every product that has **never** appeared in an order line.

*Expected:* `Unsold Monitor`.

Now deliberately write the **`NOT IN`** version too:

```sql
SELECT name FROM products
WHERE product_id NOT IN (SELECT product_id FROM order_items);
```

It happens to work here (no NULL `product_id`s). In a comment, explain what would break it: **if `order_items.product_id` contained a single NULL**, `NOT IN` would return **zero** rows. State why `NOT EXISTS` does not have that problem.

### 4. EXISTS — customers who have ordered

List the `full_name` of every customer who has placed **at least one** order, using a correlated `EXISTS`.

*Expected:* everyone **except** `Hana Kim` (customer 8 never ordered).

### 5. Correlated scalar subquery — order count per customer

For every customer, show their name and the number of orders they've placed, computed with a **correlated scalar subquery** in the `SELECT` list. Customers with no orders should show `0`.

*Hint:* `(SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id)`.

*Expected (partial):* Ana Ruiz 2, Chen Wei 2, Elif Demir 2, Hana Kim 0.

### 6. Correlated comparison — above-average customers

List customers whose order count is **greater than the average order count per customer**. The average is `total orders ÷ distinct customers-who-ordered`.

*Expected:* `Ana Ruiz`, `Chen Wei`, `Elif Demir` (2 orders each; the average is 10 ÷ 7 ≈ 1.43).

### 7. Correlated MAX — each customer's latest order

For every customer who has ordered, show their name and the date of their **most recent** order, using a correlated subquery with `MAX(order_date)`.

### 8. ANY / ALL — priced above an entire category

List products whose `price` is greater than **every** product in category 5 (Accessories), using `> ALL (subquery)`.

*Hint:* the max price in category 5 is 199.00, so you're really asking for products over 199.

## Done when…

- [ ] All eight queries run and tasks 2, 3, 4, 5, 6 match the expected values.
- [ ] Task 3 includes both the `NOT EXISTS` and `NOT IN` versions **plus** the one-sentence explanation of the NULL trap.
- [ ] You can point at each subquery and say "independent" (runs once) or "correlated" (runs per outer row), and justify it.
- [ ] Task 6 uses a subquery for the average — not a hard-coded `1.43`.

## Stretch

- Rewrite task 5 as a `LEFT JOIN … GROUP BY` instead of a correlated subquery. Which reads better? Which do you *expect* to be faster, and why? (You'll confirm with `EXPLAIN` in Week 7.)
- Write a **derived table** (subquery in `FROM`) that pre-aggregates revenue per customer, then join it back to `customers` to list name + revenue, including the zero for Hana Kim (hint: `LEFT JOIN` + `COALESCE`).
- Find every customer who has ordered a product from category 4 (Laptops) using `EXISTS` with a two-table correlated subquery.
