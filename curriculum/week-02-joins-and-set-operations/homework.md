# Week 2 — Homework

Six problems, ~5.5 hours total. All run against the `crunch_shop` seed database
(load it via [exercises/README.md](./exercises/README.md)). Commit each to your
portfolio under `c33-week-02/homework/` as a `.sql` file with your answers and a
short comment on each.

---

## Problem 1 — Join-type row-count predictions (45 min)

*Before running anything*, predict the row count of each query in a comment, then
run it and record the actual. For each mismatch, explain what you got wrong.

```sql
-- a
SELECT * FROM customers c JOIN orders o ON o.customer_id = c.customer_id;
-- b
SELECT * FROM customers c LEFT JOIN orders o ON o.customer_id = c.customer_id;
-- c
SELECT * FROM orders o JOIN order_items oi ON oi.order_id = o.order_id;
-- d
SELECT * FROM customers c CROSS JOIN regions r;
-- e
SELECT * FROM products p LEFT JOIN categories cat ON cat.category_id = p.category_id;
```

**Acceptance:** a comment with predicted vs actual for all five, and a note on
every miss. *(This trains the instinct that prevents Cartesian-explosion bugs.)*

---

## Problem 2 — The three-way question (60 min)

Answer each with one query:

1. Every order line with `order_id`, customer name, product name, and `line_total`
   (`quantity * unit_price`).
2. Total `line_total` **per order** (`GROUP BY o.order_id` — a small Week-3
   preview; `SUM(oi.quantity * oi.unit_price)`).
3. The single order with the highest total (`ORDER BY … DESC LIMIT 1`).

**Acceptance:** three queries, outputs pasted as comments. Note which order is the
biggest and its total.

---

## Problem 3 — Outer joins and the NULLs they make (60 min)

1. All customers, with a `COALESCE`'d region name (`'(no region)'` when null).
2. All products, with a `COALESCE`'d category name and supplier name.
3. Take query 1 and turn it into the **anti-join** "customers with no region" by
   testing the right-side key `IS NULL`. Which customer?
4. Deliberately reproduce the "`WHERE` demotes the `LEFT JOIN`" bug from Lecture 1,
   then fix it by moving the predicate into `ON`. Show both, with row counts.

**Acceptance:** four queries; a one-sentence explanation of the bug in problem 4.

---

## Problem 4 — Anti-joins, phrased twice (50 min)

For each, write it **both** as `NOT EXISTS` and as `LEFT JOIN … IS NULL`, and
confirm the two return identical rows:

1. Products never ordered. *(Should be two products — don't miss the second.)*
2. Employees who never took an order. *(Should be two — don't miss the CEO.)*
3. Suppliers with no products.

Then rewrite problem 4.3 as an `EXCEPT` on the supplier id. Which of the three
phrasings would you ship, and why?

**Acceptance:** each question in three forms where asked; a note on the shipping
choice. **Do not** use `NOT IN` — and in a comment, explain the `NULL` hazard
that rules it out.

---

## Problem 5 — Set operations (45 min)

1. All region IDs used by a customer **or** a supplier (`UNION`).
2. Region IDs used by **both** (`INTERSECT`).
3. Region IDs used by a supplier but **no** customer (`EXCEPT`) — name the region.
4. A single labeled feed of all parties: `'customer'`/`'supplier'` tag + name,
   sorted, using `UNION ALL`. Explain in a comment why `UNION ALL`, not `UNION`.

**Acceptance:** four queries with outputs. State the region from problem 5.3.

---

## Problem 6 — Reflection (30 min)

`reflection.md`, 250–350 words:

1. Which join type was least intuitive coming in, and what made it click?
2. Describe, in your own words, the difference between combining tables with a
   **join** vs stacking rows with a **set operation**. Give one question best
   answered by each.
3. What's still fuzzy? Be specific — "outer joins" is too broad; "why a `WHERE`
   on the right table breaks a `LEFT JOIN`" is useful.
4. One habit you want to carry into Week 3 (aggregation).

---

## Time budget

| Problem | Time |
|--------:|----:|
| 1 | 45 min |
| 2 | 1 h |
| 3 | 1 h |
| 4 | 50 min |
| 5 | 45 min |
| 6 | 30 min |
| **Total** | **~5.2 h** |

After homework, finish the [mini-project](./mini-project/README.md) and take the
[quiz](./quiz.md).
