# Lecture 2 — Self-Joins, Cross Joins, Multi-Table Joins & Anti-Joins

> **Duration:** ~2 hours. **Outcome:** You can join a table to itself, use `CROSS JOIN` deliberately, chain three-or-more tables into one query, and express "rows with no match" as an anti-join two different ways — while keeping the query readable.

Lecture 1 gave you the four join types and the `ON`/`USING` machinery. This lecture is about the *shapes* joins take in real work: a table joined to itself, the deliberate full product, long chains across a schema, and the surprisingly common question "which rows have **no** partner?" None of these need new keywords — they recombine what you already know.

## 1. Self-joins — a table joined to itself

A **self-join** joins a table to *itself*. It sounds exotic; it's just an inner or outer join where both sides are the same table, distinguished by aliases. You need it whenever a table refers to *its own* rows — the textbook case is an org chart, where each employee points at their manager, who is another employee.

In `crunch_shop`, `employees.manager_id` points back at `employees.employee_id`. To show each employee beside their manager's *name*, join `employees` to `employees`:

```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
JOIN employees m ON m.employee_id = e.manager_id
ORDER BY e.employee_id;
```

The two aliases `e` (the employee) and `m` (the manager) are what make this work — to SQL they are two independent copies of the table. Result:

```
 employee | manager
----------+---------
 Ben      | Ada
 Cy       | Ada
 Dot      | Ben
 Eve      | Cy
 Finn     | Ben
```

Ada is **missing** — she's the CEO, her `manager_id` is `NULL`, and an inner self-join drops her. To keep her, make it a `LEFT JOIN` (preserve `e`), and she appears with a `NULL` manager:

```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON m.employee_id = e.manager_id
ORDER BY e.employee_id;
```

```
 employee | manager
----------+---------
 Ada      | NULL      ← the CEO, kept by LEFT JOIN
 Ben      | Ada
 ...
```

Self-joins aren't only for hierarchies. Any "compare rows of the same table to each other" question is a self-join: find pairs of customers in the same region, find products cheaper than some other product, find two orders by the same customer on the same day. The trick is always: **alias the table twice, then write the `ON` condition that relates the two copies.**

```sql
-- Pairs of DIFFERENT customers who share a region (each pair once).
SELECT a.name AS customer_a, b.name AS customer_b, a.region_id
FROM customers a
JOIN customers b
  ON a.region_id = b.region_id      -- same region
 AND a.customer_id < b.customer_id  -- different rows; < avoids dupes + self-pairs
ORDER BY a.region_id;
```

The `a.customer_id < b.customer_id` trick is worth memorizing. Without it you'd get every customer paired with itself, and every pair twice (A-B and B-A). The strict `<` keeps each unordered pair exactly once.

## 2. `CROSS JOIN` — when you actually want the full product

Lecture 1 used `CROSS JOIN` to *explain* joins, then filtered it away. But sometimes the full Cartesian product is exactly the answer. `CROSS JOIN` has no `ON` clause — it pairs every left row with every right row.

The canonical use is **generating a dense grid** — every combination that *should* exist, so you can find the ones that don't. "Every region paired with every category" (a 4×3 = 12-row template for a coverage report):

```sql
SELECT r.name AS region, cat.name AS category
FROM regions r
CROSS JOIN categories cat
ORDER BY r.name, cat.name;   -- 12 rows: all region×category combinations
```

You'd then `LEFT JOIN` real sales onto that grid so that combinations with *zero* sales still show up as a row (with `0`), instead of silently vanishing — a technique called **densification**, which you'll meet again in Week 8's time-series work.

Two warnings about `CROSS JOIN`:

1. **It multiplies.** 1,000 × 1,000 = a million rows. An *accidental* cross join — usually a missing or wrong `ON` — is the classic "why is my query returning 40 million rows and my sums are 100× too big" bug. If a result is suspiciously huge, check that every join has a real condition.
2. **`FROM a, b` is an implicit cross join.** The old comma syntax `FROM customers, orders` with the match in `WHERE` is a `CROSS JOIN` in disguise. It works, but forget one `WHERE` predicate and you've got a silent explosion. Prefer explicit `JOIN … ON`; the condition sits right next to the join where you can't lose it.

## 3. Multi-table joins — chaining three or more tables

Nothing stops you at two tables. You chain joins, each `JOIN … ON` adding one more table to the running result. Reading a full order line — customer, product, category — takes four tables because `order_items` sits between orders and products:

```sql
SELECT
  o.order_id,
  c.name         AS customer,
  p.name         AS product,
  cat.name       AS category,
  oi.quantity,
  oi.unit_price
FROM orders o
JOIN customers   c   ON c.customer_id  = o.customer_id
JOIN order_items oi  ON oi.order_id    = o.order_id
JOIN products    p   ON p.product_id   = oi.product_id
JOIN categories  cat ON cat.category_id = p.category_id
ORDER BY o.order_id, p.name;
```

Read it as a pipeline: start with `orders`, attach the `customer`, expand to one row per `order_item`, attach each item's `product`, attach that product's `category`. Each join answers "and also give me the …".

### How to read (and write) a multi-table join

1. **Pick a driving table** — the "grain" of your result. Here it's `order_items` (one row per line item), reached through `orders`. Decide up front what one output row *means*.
2. **Join outward along the foreign keys.** Each table connects to the running set through a key that already exists. If you can't state the `ON` condition, the two tables aren't directly related — you need an intermediate table (that's why `order_items` is there: orders and products have no direct link).
3. **Keep the ON conditions complete.** Every `JOIN` after the first should have an `ON` that ties the new table to something already in the query. A `JOIN` whose `ON` doesn't reference the existing tables is an accidental cross join.
4. **Watch the grain change.** The moment `order_items` enters, one order becomes several rows. If a later step sums `unit_price`, that's fine; if it sums `orders.some_total`, you'll double-count. Know your grain.

### `INNER` vs `LEFT` in a chain

Mixing join types in one chain is normal and meaningful. Product 6 (Gizmo-Pro) has **no category**. With an `INNER JOIN` to `categories`, any order line for Gizmo-Pro would vanish. If the report must show every line, make just that last hop a `LEFT JOIN`:

```sql
... JOIN products p ON p.product_id = oi.product_id
LEFT JOIN categories cat ON cat.category_id = p.category_id   -- keep no-category products
```

Once one join in the chain is `LEFT`, be careful that a *later* `INNER JOIN` or a `WHERE` doesn't drop the rows you just preserved — the same demotion trap from Lecture 1, now hiding in a longer query.

## 4. Anti-joins — "rows with no match"

A whole category of questions is negative: customers who **never** ordered, products **never** sold, suppliers with **no** products. This is the **anti-join** — return rows from A that have *no* matching row in B. SQL has no `ANTI JOIN` keyword; you express it. There are two standard, correct ways, plus one common mistake.

### 4.1 `LEFT JOIN … WHERE right IS NULL`

`LEFT JOIN` keeps every left row, filling `NULL` where there's no match. So the unmatched rows are exactly the ones where the right side came back `NULL`. Filter for that:

```sql
-- Customers who never placed an order.
SELECT c.customer_id, c.name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL;      -- no order matched → unmatched customer
```

Result: `7 | Gizmo Inc`. The `IS NULL` must be on a right-table column that could **never** legitimately be `NULL` for a real match — a primary key (`o.order_id`) is the safe choice. Testing `IS NULL` on a nullable column would misfire.

### 4.2 `NOT EXISTS` (usually the best choice)

`NOT EXISTS` reads closer to English — "give me each customer for whom there does not exist an order" — and the planner handles it cleanly:

```sql
SELECT c.customer_id, c.name
FROM customers c
WHERE NOT EXISTS (
  SELECT 1
  FROM orders o
  WHERE o.customer_id = c.customer_id
);
```

Same result. `SELECT 1` is idiomatic — `EXISTS` only cares *whether* the subquery returns any row, never *what* it returns, so you select a constant.

### 4.3 The `NOT IN` trap with `NULL`

There's a third form, `NOT IN`, that looks equivalent and has a vicious edge case:

```sql
-- DANGER: breaks if the subquery can return a NULL.
SELECT c.name
FROM customers c
WHERE c.customer_id NOT IN (SELECT o.customer_id FROM orders o);
```

If the subquery's column ever contains a `NULL`, `NOT IN` returns **no rows at all** — because `x NOT IN (…, NULL)` evaluates to `NULL` (unknown), never true. `orders.customer_id` happens to be non-null here so it works today, but the moment a nullable column feeds a `NOT IN`, the query silently returns nothing. **Prefer `NOT EXISTS`**, which handles `NULL`s correctly and doesn't depend on this hidden assumption.

### Anti-join vs semi-join

The positive mirror of the anti-join is the **semi-join**: rows from A that *do* have a match in B, but without multiplying or pulling B's columns. That's `EXISTS` (or `IN`):

```sql
-- Customers who HAVE ordered (semi-join) — each customer once, no fan-out.
SELECT c.name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);
```

Why not a plain `INNER JOIN`? Because a customer with 3 orders would appear 3 times; you'd need `DISTINCT`. `EXISTS` gives you the "at least one match" semantics with no duplicates. Keep the pair straight:

| Question | Tool |
|----------|------|
| Rows in A **with** a match in B (once each) | semi-join — `EXISTS` / `IN` |
| Rows in A with **no** match in B | anti-join — `NOT EXISTS` / `LEFT JOIN … IS NULL` |
| Rows in A joined **to** B's columns (may multiply) | `INNER JOIN` |

## 5. Join order & readability

The SQL you write is a *declaration* of what you want; the planner is free to execute the joins in whatever physical order is fastest (you'll watch it do this in Week 7). So for **inner joins**, the order you list tables does not change the result — `a JOIN b JOIN c` returns the same rows as `c JOIN b JOIN a`. For **outer joins**, order *does* matter, because "which side is preserved" is baked into left/right.

Since order rarely affects correctness for inner joins, spend your ordering budget on **the human reader**:

- **Start from the table that defines the grain** — the thing you're really counting/listing (orders, or order items). Everything else hangs off it.
- **Join in the direction the foreign keys point**, so each `ON` references a table already above it. A reader can follow the chain top-down.
- **Align the `ON` conditions** and keep one join per line. A 5-table join is readable when it's a neat ladder and unreadable as a run-on.
- **Alias consistently** — a short, memorable alias per table (`o`, `c`, `oi`, `p`, `cat`). Reuse the same aliases across the whole codebase and queries start to feel familiar.

```sql
-- Readable 5-table ladder: grain = order_items, joined outward.
SELECT o.order_id, c.name AS customer, e.name AS rep,
       p.name AS product, oi.quantity
FROM order_items oi
JOIN orders    o ON o.order_id     = oi.order_id
JOIN customers c ON c.customer_id  = o.customer_id
JOIN employees e ON e.employee_id  = o.employee_id
JOIN products  p ON p.product_id   = oi.product_id
ORDER BY o.order_id, p.name;
```

## 6. Check yourself

- Why does a self-join need two aliases? What breaks if you omit them?
- What does `a.id < b.id` accomplish in a self-join of pairs?
- Give one legitimate use of `CROSS JOIN` and one symptom of an *accidental* one.
- In a 4-table chain, `order_items` sits between `orders` and `products`. Why can't you join `orders` straight to `products`?
- Write "products never ordered" two ways: `LEFT JOIN … IS NULL` and `NOT EXISTS`.
- Why can `NOT IN (subquery)` return zero rows unexpectedly, and what should you use instead?
- Semi-join vs `INNER JOIN` — what problem does `EXISTS` avoid?

When those are automatic, do [Exercise 3](../exercises/exercise-03-self-joins-and-set-ops.md) and [Challenge 2](../challenges/challenge-02-find-the-missing-matches.md), then read [Lecture 3](./03-set-operations.md).

## Further reading

- **PostgreSQL — subquery expressions (`EXISTS`, `IN`, `NOT IN`):** <https://www.postgresql.org/docs/16/functions-subquery.html>
- **Markus Winand — "The SQL anti-join" (Use The Index, Luke):** <https://use-the-index-luke.com/sql/join>
- **Why `NOT IN` with NULL bites — worked example:** <https://wiki.postgresql.org/wiki/Don%27t_Do_This#Don.27t_use_NOT_IN>
