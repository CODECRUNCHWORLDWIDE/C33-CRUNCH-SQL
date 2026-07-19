# Week 3 — Quiz

Fifteen questions on aggregation, subqueries, and CTEs. Lectures closed. Aim for 13/15 before Week 4. Some answers require reading a query against the `crunch_shop` seed data.

---

**Q1.** What does an aggregate function do to its input rows?

- A) Returns one output row per input row.
- B) Collapses a set of rows into a single value.
- C) Sorts the rows.
- D) Removes duplicate rows.

---

**Q2.** In a query with `GROUP BY country`, which columns may appear un-aggregated in the `SELECT` list?

- A) Any column in the table.
- B) Only `country`.
- C) Only columns inside an aggregate.
- D) `country` plus any column also in the same table.

---

**Q3.** `COUNT(*)` returns 10 for a group but `COUNT(discount)` returns 6. What is true?

- A) There are 4 rows where `discount` is NULL.
- B) There are 6 rows where `discount` is NULL.
- C) The query is invalid.
- D) `discount` has 10 distinct values.

---

**Q4.** Which condition **must** go in `HAVING` rather than `WHERE`?

- A) `status = 'paid'`
- B) `country = 'US'`
- C) `SUM(quantity * unit_price) > 500`
- D) `order_date >= '2024-06-01'`

---

**Q5.** `COUNT(*) FILTER (WHERE status = 'paid')` is equivalent to:

- A) `COUNT(status = 'paid')`
- B) `SUM(CASE WHEN status = 'paid' THEN 1 ELSE 0 END)`
- C) `COUNT(DISTINCT status)`
- D) `WHERE status = 'paid'` applied to the whole query

---

**Q6.** Which grouping construct produces `(a,b)`, `(a)`, `(b)`, and `()` — every combination?

- A) `ROLLUP (a, b)`
- B) `GROUPING SETS ((a,b))`
- C) `CUBE (a, b)`
- D) `GROUP BY a, b`

---

**Q7.** Why can `WHERE customer_id NOT IN (SELECT customer_id FROM orders)` return zero rows unexpectedly?

- A) `NOT IN` is not valid SQL.
- B) If the subquery yields any NULL, the whole test becomes UNKNOWN and no row qualifies.
- C) `orders` is empty.
- D) `NOT IN` only works on numbers.

---

**Q8.** A subquery is **correlated** when it:

- A) Returns exactly one row.
- B) Appears in the `FROM` clause.
- C) References a column from the outer query.
- D) Uses `EXISTS`.

---

**Q9.** Why is `EXISTS (SELECT 1 …)` conventionally written with `1`?

- A) `EXISTS` requires the literal `1`.
- B) It's faster than `SELECT *` by a large margin.
- C) `EXISTS` only tests whether a row exists; the projected value is irrelevant, so `1` documents that.
- D) `1` is the row count.

---

**Q10.** In a chained `WITH a AS (…), b AS (…)`, which reference is legal?

- A) `a` may read from `b`.
- B) `b` may read from `a`.
- C) Neither may read from the other.
- D) Both may read from each other freely.

---

**Q11.** A recursive CTE is guaranteed to terminate when:

- A) It uses `UNION` instead of `UNION ALL`.
- B) The recursive term eventually returns no new rows (e.g., via a depth cap or an acyclic structure).
- C) It has an `ORDER BY`.
- D) The anchor returns exactly one row.

---

**Q12.** Reading the seed data: how many customers have placed **two or more** orders?

- A) 2
- B) 3
- C) 4
- D) 5

---

**Q13.** What did PostgreSQL 12 change about CTEs?

- A) It removed recursive CTEs.
- B) It made single-reference, non-recursive CTEs inlinable instead of always an optimisation fence.
- C) It made all CTEs materialise.
- D) Nothing; CTEs are unchanged since PostgreSQL 8.

---

**Q14.** `SUM(amount)` over a group that contains **no** rows returns:

- A) `0`
- B) `NULL`
- C) An error.
- D) The number of NULLs.

---

**Q15.** In the query `... GROUP BY ROLLUP (country, status)`, a result row where `status IS NULL` (and it is not real missing data) represents:

- A) A bug.
- B) A subtotal across all statuses for that country.
- C) A cancelled order.
- D) The grand total.

---

## Answer key

<details>
<summary>Reveal after attempting</summary>

1. **B** — an aggregate collapses a set of rows into one value.
2. **B** — only the grouping column(s); everything else must be aggregated. (SQLite would *allow* more but pick an arbitrary value — don't rely on it.)
3. **A** — `COUNT(col)` skips NULLs, so 10 − 6 = 4 NULL discounts.
4. **C** — only the aggregate condition must go in `HAVING`; the others are row filters that belong in `WHERE`.
5. **B** — `FILTER` restricts that one aggregate to matching rows, exactly like the `CASE`/`SUM` idiom.
6. **C** — `CUBE` produces every combination of the grouping columns.
7. **B** — the `NOT IN` / NULL trap: one NULL makes the predicate UNKNOWN for every row. Use `NOT EXISTS`.
8. **C** — correlation means the subquery references an outer column and (conceptually) re-runs per outer row.
9. **C** — `EXISTS` ignores the projected value; `1` signals "value doesn't matter." (Performance is identical to `SELECT *`.)
10. **B** — a later CTE may reference earlier ones; not the reverse (non-recursive case).
11. **B** — termination comes from the recursive term running dry, e.g., an acyclic structure or a depth cap.
12. **B** — 3 (customers 1, 3, and 5 each have two orders).
13. **B** — PostgreSQL 12 inlines single-reference, non-recursive, side-effect-free CTEs, with `MATERIALIZED`/`NOT MATERIALIZED` to override.
14. **B** — `SUM` of an empty set is `NULL`. `COALESCE(SUM(x), 0)` if you want 0.
15. **B** — a `ROLLUP` subtotal row; the rolled-up column comes back NULL. Use `GROUPING()` to label it.

</details>

**Scoring:** 13+ correct → move to Week 4. 10–12 → re-read the lecture sections behind your misses. <10 → re-read all three lectures and redo Exercise 1–3.
