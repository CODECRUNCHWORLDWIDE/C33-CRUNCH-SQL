# Lecture 1 — Aggregation, GROUP BY, and HAVING

> **Duration:** ~2 hours. **Outcome:** You can collapse many rows into one number with the aggregate functions, split a table into groups with `GROUP BY`, filter those groups with `HAVING`, count a *subset* of each group with `FILTER`, and produce subtotals with `GROUPING SETS` / `ROLLUP` / `CUBE` — and you know exactly which of these your engine supports.

Weeks 1 and 2 kept every input row visible in the output: filter, sort, join, one row in, (up to) one row out. Aggregation is the first tool that **destroys** rows on purpose. You feed it a thousand order lines and it hands back a single average. Understanding *when* rows collapse — and which columns survive the collapse — is the entire mental model. Get that right and `HAVING`, `FILTER`, and grouping sets are just refinements.

Every query in this lecture runs against the course seed database, `crunch_shop` (schema and seed SQL are in `exercises/README.md`). If you have not loaded it yet, do that first — you will type these queries, not just read them.

## 1. Why aggregation exists

A relational table is a *set of rows*. Most business questions are not about a row — they are about a **collection** of rows:

- "How many orders did we take last month?" — one number from thousands of rows.
- "What is the average order value per country?" — one number *per group*.
- "Which products have never sold?" — a comparison between a set and its subset.

SQL answers these with **aggregate functions**: functions that take a whole column of values and return a single scalar. The instant you put an aggregate in your `SELECT`, the query stops returning one row per input row and starts returning one row per *group* (and with no `GROUP BY`, exactly one row for the entire table).

That is the rule to burn in: **an aggregate turns a set of rows into one value.** Everything else follows.

## 2. The five core aggregates (plus the ones you'll actually reach for)

| Function | Returns | NULL handling |
|----------|---------|---------------|
| `COUNT(*)` | number of rows in the group | counts every row, NULLs included |
| `COUNT(col)` | number of rows where `col IS NOT NULL` | **skips NULLs** — a classic trap |
| `COUNT(DISTINCT col)` | number of distinct non-NULL values | skips NULLs |
| `SUM(col)` | total of the values | ignores NULLs; returns NULL for an empty group |
| `AVG(col)` | mean of the values | ignores NULLs (divides by the count of non-NULLs) |
| `MIN(col)` / `MAX(col)` | smallest / largest | ignore NULLs; work on text and dates too |

Three behaviours trip up almost everyone:

1. **`COUNT(*)` vs `COUNT(col)`.** `COUNT(*)` counts rows. `COUNT(status)` counts rows where `status` is not NULL. If a column is nullable, these differ — and the difference is often the answer to "how many are missing?"
2. **`AVG` ignores NULLs in the denominator.** `AVG(discount)` over ten rows where three discounts are NULL divides the sum by **seven**, not ten. If you meant "treat missing as zero," write `AVG(COALESCE(discount, 0))`.
3. **`SUM` of an empty set is NULL, not 0.** Wrap it: `COALESCE(SUM(amount), 0)` when a zero is more honest than a blank.

A first taste — the whole table as one group:

```sql
SELECT COUNT(*)                         AS order_count,
       COUNT(DISTINCT customer_id)      AS distinct_customers,
       MIN(order_date)                  AS first_order,
       MAX(order_date)                  AS last_order
FROM orders;
```

No `GROUP BY`, one aggregate-bearing `SELECT` → exactly one row back, summarising all of `orders`.

## 3. GROUP BY: splitting the table before you aggregate

`GROUP BY` says: *before aggregating, partition the rows into buckets that share the same value(s), then compute each aggregate once per bucket.*

```sql
SELECT country,
       COUNT(*)        AS customers,
       MIN(signup_date) AS earliest_signup
FROM customers
GROUP BY country
ORDER BY customers DESC;
```

Rows with `country = 'US'` form one bucket, `'CA'` another, and so on. Each bucket yields one output row.

### The single most important GROUP BY rule

Every column in the `SELECT` list must be **either** inside an aggregate **or** named in the `GROUP BY`. Nothing else is allowed.

```sql
-- WRONG: full_name is neither aggregated nor grouped
SELECT country, full_name, COUNT(*)
FROM customers
GROUP BY country;
```

PostgreSQL rejects this with `column "customers.full_name" must appear in the GROUP BY clause or be used in an aggregate function`. The reason is not pedantry: a group can contain a hundred different `full_name` values, so which one would the engine print? The query is ambiguous, so it is illegal.

> **Engine trap:** SQLite (and old MySQL) *will* run the query above and silently pick an arbitrary `full_name` from the group. That is a footgun, not a feature — the value is unpredictable. Write GROUP BY the Postgres-strict way and both engines agree.

### You can group by more than one column

```sql
SELECT o.status,
       c.country,
       COUNT(*)     AS orders,
       SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN customers   c  ON c.customer_id = o.customer_id
JOIN order_items oi ON oi.order_id  = o.order_id
GROUP BY o.status, c.country
ORDER BY revenue DESC;
```

The bucket key is now the *pair* `(status, country)`. One output row per distinct pair that actually occurs in the data.

### Grouping by an expression

You can group by a computed value — just repeat the expression (Postgres also lets you group by a `SELECT`-list alias or ordinal, but repeating the expression is the portable habit):

```sql
SELECT date_trunc('month', order_date) AS month,
       COUNT(*)                        AS orders
FROM orders
GROUP BY date_trunc('month', order_date)
ORDER BY month;
```

In SQLite the monthly bucket is `strftime('%Y-%m', order_date)` — same idea, different function.

## 4. WHERE vs HAVING: filter rows, then filter groups

This is the distinction the whole lecture builds toward.

- **`WHERE` filters rows** *before* grouping. It cannot see aggregates — the groups don't exist yet.
- **`HAVING` filters groups** *after* aggregating. It can (and usually should) reference aggregates.

```sql
SELECT c.country,
       COUNT(*)                          AS paid_orders,
       SUM(oi.quantity * oi.unit_price)  AS revenue
FROM orders o
JOIN customers   c  ON c.customer_id = o.customer_id
JOIN order_items oi ON oi.order_id  = o.order_id
WHERE o.status = 'paid'                 -- row filter: drop non-paid lines FIRST
GROUP BY c.country
HAVING SUM(oi.quantity * oi.unit_price) > 500   -- group filter: keep big-revenue countries
ORDER BY revenue DESC;
```

Read it in execution order, which is *not* the written order:

| Step | Clause | What happens |
|-----:|--------|--------------|
| 1 | `FROM` / `JOIN` | assemble the rows |
| 2 | `WHERE` | drop rows that fail the row predicate (no aggregates allowed) |
| 3 | `GROUP BY` | partition surviving rows into groups |
| 4 | `HAVING` | drop groups that fail the group predicate (aggregates allowed) |
| 5 | `SELECT` | compute the output columns |
| 6 | `ORDER BY` | sort the result |

Two consequences worth memorising:

1. **Put a condition in `WHERE` whenever it looks at a plain column, not an aggregate.** `WHERE status = 'paid'` filters early, so the engine aggregates fewer rows — it is both correct and faster. Only conditions that reference an aggregate (`SUM(...) > 500`) *must* go in `HAVING`.
2. **`HAVING` without `GROUP BY` is legal.** It filters the single implicit whole-table group: `SELECT COUNT(*) FROM orders HAVING COUNT(*) > 100` returns the count only if there are more than 100 orders.

## 5. FILTER: a per-aggregate WHERE

Often you want several counts over *different* subsets in one pass — "paid orders, refunded orders, and cancelled orders, side by side." The clumsy way is `CASE` inside `SUM`. The clean, standard way is the `FILTER (WHERE …)` clause:

```sql
SELECT c.country,
       COUNT(*)                                    AS all_orders,
       COUNT(*) FILTER (WHERE o.status = 'paid')      AS paid,
       COUNT(*) FILTER (WHERE o.status = 'refunded')  AS refunded,
       SUM(oi.quantity * oi.unit_price)
              FILTER (WHERE o.status = 'paid')      AS paid_revenue
FROM orders o
JOIN customers   c  ON c.customer_id = o.customer_id
JOIN order_items oi ON oi.order_id  = o.order_id
GROUP BY c.country;
```

`FILTER` restricts *that one aggregate* to the rows matching its predicate, without touching the other aggregates. It is far more readable than the pre-`FILTER` idiom:

```sql
-- the old CASE trick — equivalent, but noisier
SUM(CASE WHEN o.status = 'paid' THEN 1 ELSE 0 END) AS paid
```

| Engine | `FILTER (WHERE …)` support |
|--------|----------------------------|
| PostgreSQL | Yes (all supported versions) |
| SQLite | Yes, 3.30+ (2019). Older builds: fall back to `CASE`. |
| MySQL | No — use the `CASE` form |

`FILTER` and `CASE` produce identical results; prefer `FILTER` where it exists.

## 6. Subtotals: GROUPING SETS, ROLLUP, CUBE

A plain `GROUP BY country, status` gives you one row per combination. But a report usually also wants **subtotals** ("all statuses within a country") and a **grand total** ("everything"). Running three separate queries and unioning them works but is ugly. SQL gives you three shorthands.

- **`GROUPING SETS`** — you list *exactly* which groupings you want.
- **`ROLLUP(a, b)`** — hierarchical subtotals: `(a, b)`, then `(a)`, then `()`. Think year → month → grand total.
- **`CUBE(a, b)`** — *every* combination: `(a, b)`, `(a)`, `(b)`, `()`.

```sql
SELECT c.country,
       o.status,
       SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN customers   c  ON c.customer_id = o.customer_id
JOIN order_items oi ON oi.order_id  = o.order_id
GROUP BY ROLLUP (c.country, o.status)
ORDER BY c.country, o.status;
```

`ROLLUP (country, status)` produces, per country, one row per status **plus** a country subtotal row (where `status` is NULL), **plus** one final grand-total row (both NULL). That NULL is how you spot a subtotal — the column that was "rolled up" comes back NULL.

Because a real NULL and a subtotal-marker NULL look identical, use the **`GROUPING()`** function to tell them apart. `GROUPING(status)` returns `1` when this row is a subtotal over `status`, `0` otherwise:

```sql
SELECT CASE WHEN GROUPING(c.country) = 1 THEN 'ALL COUNTRIES' ELSE c.country END AS country,
       CASE WHEN GROUPING(o.status)  = 1 THEN 'ALL STATUSES'  ELSE o.status  END AS status,
       SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN customers   c  ON c.customer_id = o.customer_id
JOIN order_items oi ON oi.order_id  = o.order_id
GROUP BY ROLLUP (c.country, o.status);
```

The three set operators, side by side:

| Written as | Groupings produced | Typical use |
|------------|--------------------|-------------|
| `GROUP BY GROUPING SETS ((a,b), (a), ())` | exactly those three | full control |
| `GROUP BY ROLLUP (a, b)` | `(a,b)`, `(a)`, `()` | hierarchy / drill-down totals |
| `GROUP BY CUBE (a, b)` | `(a,b)`, `(a)`, `(b)`, `()` | cross-tab, every margin |

> **Engine trap:** `GROUPING SETS`, `ROLLUP`, `CUBE`, and `GROUPING()` are **PostgreSQL-only** among our two engines — SQLite does not implement them. On SQLite, emulate a rollup by `UNION ALL`-ing a detail query with a subtotal query. This is exactly the kind of gap that makes "know your engine" a real skill, not a slogan.

## 7. A word on DISTINCT inside aggregates

`COUNT(DISTINCT customer_id)` counts unique customers; `SUM(DISTINCT amount)` sums each distinct amount once (rarely what you want). `DISTINCT` inside an aggregate is evaluated *within the group*. It is the right tool for "how many *different* products did each customer buy" and the wrong tool for almost anything involving `SUM` — reach for it deliberately.

```sql
SELECT o.customer_id,
       COUNT(*)                       AS line_items,
       COUNT(DISTINCT oi.product_id)  AS distinct_products
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY o.customer_id;
```

## 8. Check yourself

- An aggregate function turns *what* into *what*? (A set of rows into one value.)
- Why must every non-aggregated `SELECT` column appear in `GROUP BY`?
- Which of these belongs in `WHERE` and which in `HAVING`: `status = 'paid'`, `SUM(amount) > 1000`, `country = 'US'`, `COUNT(*) >= 3`?
- `COUNT(*)` returns 10 but `COUNT(discount)` returns 7. What do you now know about the data?
- Rewrite `SUM(CASE WHEN status='paid' THEN amount ELSE 0 END)` using `FILTER`.
- You run a `ROLLUP(country, status)` report and see a row with `status = NULL`. What is that row, and how do you label it reliably?
- Which of `FILTER`, `ROLLUP`, and `GROUPING SETS` work on SQLite? Which do not?

If you can answer all seven without scrolling up, load the seed database and move to Lecture 2.

## Further reading

- **PostgreSQL 16 — Aggregate Functions:** <https://www.postgresql.org/docs/16/functions-aggregate.html>
- **PostgreSQL 16 — GROUPING SETS, CUBE, ROLLUP:** <https://www.postgresql.org/docs/16/queries-table-expressions.html#QUERIES-GROUPING-SETS>
- **SQLite — Aggregate Functions:** <https://www.sqlite.org/lang_aggfunc.html>
