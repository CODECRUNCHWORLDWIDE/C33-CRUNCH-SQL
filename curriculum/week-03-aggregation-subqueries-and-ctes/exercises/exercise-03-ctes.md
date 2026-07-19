# Exercise 3 — CTEs (including one recursive)

**Goal:** Refactor a nested query into readable chained CTEs, then write a recursive CTE that walks a hierarchy — both a category tree and an org chart.

**Estimated time:** 55 minutes.

## Before you start

Seed database loaded. This exercise uses `categories` and `employees` (the two self-referencing tables) in addition to the store tables. Answers into `solutions.sql`.

## Part A — Chained CTEs

### 1. One CTE — paid revenue per customer

Write a query with a single CTE `paid_lines` that selects `customer_id` and `quantity * unit_price AS line_total` for **paid** orders only, then in the main query returns each customer's total revenue, highest first.

*Expected (partial):* customer 1 → 1509.00, customer 3 → 319.00, customer 6 → 244.00.

### 2. Chain three CTEs — above-average customers

Build a three-stage pipeline:

- `paid_lines` — as in task 1.
- `per_customer` — group `paid_lines` to get `revenue` per `customer_id`.
- `overall` — a one-row CTE holding `AVG(revenue)` across `per_customer`.

Then join `per_customer` to `overall` and to `customers`, and return the name + revenue of every customer whose revenue is **above the average paid revenue**, highest first.

Name each CTE for *what it holds*, not `t1`/`t2`. This is the exact decomposition the mini-project rewards.

### 3. Readability check

Take your task-2 answer and, in a comment, note which stage you would inspect first if the final numbers looked wrong — and why the named stages make that debugging easy compared to one giant nested subquery.

## Part B — Recursive CTEs

### 4. Category subtree — walk *down*

Write a `WITH RECURSIVE` query that lists every category in the subtree rooted at **`Electronics`**, carrying a `depth` column (0 at Electronics). Render the name indented by depth (`repeat('  ', depth) || name` in Postgres; `printf` or `||` in SQLite).

*Expected rows (name, depth):* `Electronics` 0, `Computers` 1, `Accessories` 1, `Laptops` 2.

### 5. Org chart — walk *up*

Write a `WITH RECURSIVE` query that, starting from employee **`Dana Ito`**, lists her entire chain of command up to the top, with a `level` column (0 at Dana).

*Expected (level, name):* 0 `Dana Ito`, 1 `Vera Lopez`, 2 `Root Boss`.

State in a comment the **one line** you would change to turn this into "list all of Root Boss's subordinates" (walk down instead of up).

### 6. Guard against runaway recursion

Add a depth cap to your task-4 query (`WHERE depth < 20` in the recursive term). Explain in a comment why you'd keep this guard even though the seed data is a clean tree with no cycles.

## Done when…

- [ ] Tasks 1, 2, 4, and 5 match the expected values.
- [ ] Task 2 uses **three** named CTEs, each referencing the previous, with descriptive names.
- [ ] Both recursive queries use `WITH RECURSIVE`, an anchor, `UNION ALL`, and a recursive term that joins back to the CTE.
- [ ] Task 6 has a working depth cap and a one-sentence justification.

## Stretch

- Extend task 4 to also print each category's **product count** by joining the recursive result to `products` (remember a parent category may have zero direct products).
- Write a recursive CTE that computes each category's *full path* as text, e.g. `All > Electronics > Computers > Laptops`, by concatenating names down the tree.
- Combine Part A and Part B: a CTE chain where the first CTE is the recursive category subtree and a later CTE aggregates revenue for products in that subtree.
