# Week 3 — Homework

Six problems, ~5–6 hours total. All run against the `crunch_shop` seed database (see `exercises/README.md`). Commit each answer to your portfolio under `c33-week-03/homework/`.

Type your queries; verify each result by hand-count on this small dataset before moving on.

---

## Problem 1 — Aggregate warm-up (45 min)

In `p1.sql`, answer each with one query:

1. Total number of orders, and the number in each status (one row per status).
2. The single highest-priced product and its price (a scalar aggregate + a subquery, or an `ORDER BY … LIMIT 1`).
3. Average order-line revenue (`quantity * unit_price`) across all order lines.
4. Per country: number of customers and their earliest signup date.

**Acceptance:** four queries, each correct against a hand count; a one-line comment on each explaining what it computes.

---

## Problem 2 — WHERE vs HAVING (45 min)

Write, in `p2.sql`, the query: *"countries whose **paid** revenue exceeds 400, listed high to low."*

Then, in comments, answer:

1. Which condition went in `WHERE` and which in `HAVING`? Why is each in the right place?
2. What happens to the result if you (wrongly) move `status = 'paid'` into `HAVING`? Try it and describe the error or wrong answer.
3. What happens if you move the revenue threshold into `WHERE`? Try it and describe the error.

**Acceptance:** the correct query plus three honest observations from actually running the broken variants.

---

## Problem 3 — FILTER and a status matrix (45 min)

In `p3.sql`, produce one row per country with: total orders, and a separate count for each of the four statuses, using `COUNT(*) FILTER (WHERE …)`.

Then add a `paid_revenue` column (revenue from paid lines only) to the same grouped query — again with `FILTER`, not a second query.

**Acceptance:** a single grouped query with the full status matrix + paid_revenue; if on old SQLite, the documented `CASE` fallback and a note that the numbers match.

---

## Problem 4 — Subquery trio (1 h)

In `p4.sql`, answer all three, each with a subquery (not a join where a subquery is asked for):

1. **Scalar:** products priced above the average product price.
2. **Correlated `EXISTS`:** customers who have ordered a product in category 4 (Laptops).
3. **`NOT EXISTS`:** products never ordered — and, in a comment, the `NOT IN` version plus one sentence on why `NOT EXISTS` is the safer default (the NULL trap).

**Acceptance:** three queries; the `EXISTS` one is genuinely correlated (references the outer customer); the NULL-trap explanation is correct.

---

## Problem 5 — CTE pipeline (1 h)

In `p5.sql`, rewrite this deliberately-ugly nested query as a chain of **named** CTEs, then confirm the output is identical:

> "For each customer with above-average paid revenue, list their name, paid revenue, and how many distinct products they bought."

Requirements:

- At least three named CTEs (`paid_lines`, `per_customer`, `overall`, …).
- No stage named `t1`/`sub`/`x`.
- A one-line comment per CTE saying what it holds.

**Acceptance:** a readable CTE chain producing the right customers; a note on which stage you'd inspect first if a number looked wrong.

---

## Problem 6 — Recursive hierarchy (1 h)

In `p6.sql`:

1. Write a recursive CTE that returns the **full category tree** starting at `All` (category 1), with a `depth` column and an indented name.
2. Write a recursive CTE that, for employee `Tara Singh`, returns her **entire management chain** up to the top, with a `level` column.
3. Add a depth guard (`WHERE depth < 20`) to query 1 and, in a comment, explain why you'd keep it even on clean data.

*Spot-check:* Tara Singh (employee 6) → Dana Ito → Vera Lopez → Root Boss (levels 0–3).

**Acceptance:** both recursive queries correct; a working depth guard; the management chain matches the spot-check.

---

## Time budget

| Problem | Time |
|--------:|----:|
| 1 | 45 min |
| 2 | 45 min |
| 3 | 45 min |
| 4 | 1 h |
| 5 | 1 h |
| 6 | 1 h |
| **Total** | **~5.25 h** |

After homework, ship the [mini-project](./mini-project/README.md) and take the [quiz](./quiz.md).
