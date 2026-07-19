# Week 2 — Quiz

Thirteen questions. Lectures closed. Aim for 11/13 before Week 3. Some reference
the `crunch_shop` seed data — keep the gaps in mind (Gizmo Inc never ordered,
Widget-X never sold, West region has no customers, Ada has no manager).

---

**Q1.** A join is best understood as:

- A) A `CROSS JOIN` followed by a `WHERE`/`ON` condition that keeps matching pairs.
- B) A way to add rows from a second table.
- C) A merge that always removes duplicates.
- D) The same thing as a `UNION`.

---

**Q2.** `customers` has 7 rows, `orders` has 7 rows. How many rows does
`SELECT * FROM customers CROSS JOIN orders` return?

- A) 7
- B) 14
- C) 49
- D) 0

---

**Q3.** Which rows does `A LEFT JOIN B` return that `A INNER JOIN B` does not?

- A) Rows of B with no match in A.
- B) Rows of A with no match in B (B's columns filled with `NULL`).
- C) Duplicate rows.
- D) None — they're identical.

---

**Q4.** Why do experienced developers avoid `NATURAL JOIN`?

- A) It's slower than `INNER JOIN`.
- B) It joins on all same-named columns implicitly, so adding a column can silently change results.
- C) It doesn't work in PostgreSQL.
- D) It can't be used with aliases.

---

**Q5.** This query returns fewer rows than expected:

```sql
SELECT c.name, o.status
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.status = 'shipped';
```

What happened?

- A) `LEFT JOIN` is broken.
- B) The `WHERE` on the right-table column drops the `NULL` rows, demoting the join to an inner join.
- C) `customers` has no `name` column.
- D) `status` values are case-sensitive.

---

**Q6.** To keep the CEO (whose `manager_id` is `NULL`) in an employee→manager
self-join, you must:

- A) Use `INNER JOIN`.
- B) Use `LEFT JOIN` preserving the employee side.
- C) Use `CROSS JOIN`.
- D) Add `WHERE manager_id IS NOT NULL`.

---

**Q7.** In a self-join to list unordered pairs of coworkers, what does
`a.employee_id < b.employee_id` accomplish?

- A) Sorts the output.
- B) Removes self-pairs and keeps each pair only once (not both orderings).
- C) Filters to managers only.
- D) Nothing — it's decorative.

---

**Q8.** Which is the safest, most portable way to write an anti-join
("A with no match in B")?

- A) `WHERE a.id NOT IN (SELECT b.a_id FROM b)`
- B) `WHERE NOT EXISTS (SELECT 1 FROM b WHERE b.a_id = a.id)`
- C) `A CROSS JOIN B`
- D) `A INNER JOIN B ... WHERE b.id IS NULL`

---

**Q9.** `x NOT IN (subquery)` can return **zero** rows unexpectedly when:

- A) The subquery is empty.
- B) The subquery returns a `NULL`, making `x NOT IN (…, NULL)` never true.
- C) `x` is an integer.
- D) You forgot an `ORDER BY`.

---

**Q10.** The difference between `UNION` and `UNION ALL` is:

- A) `UNION ALL` removes duplicates; `UNION` keeps them.
- B) `UNION` removes duplicates (and is slower); `UNION ALL` keeps all rows.
- C) They're identical.
- D) `UNION ALL` only works in SQLite.

---

**Q11.** For a set operation to be valid, the two `SELECT`s must:

- A) Query the same table.
- B) Have the same number of columns with compatible types.
- C) Have identical column names.
- D) Both include an `ORDER BY`.

---

**Q12.** `{suppliers' regions} EXCEPT {customers' regions}` in `crunch_shop`
returns:

- A) Every region.
- B) An empty set.
- C) Region 4 (West) — it has a supplier but no customer.
- D) Regions 1, 2, 3.

---

**Q13.** You need "customers who never ordered, **with their name and region**."
The best tool is:

- A) `EXCEPT` on the customer id (it returns only the id).
- B) An anti-join (`NOT EXISTS` / `LEFT JOIN … IS NULL`), because you need columns beyond the key.
- C) A `CROSS JOIN`.
- D) `UNION ALL`.

---

## Answer key

<details>
<summary>Reveal after attempting</summary>

1. **A** — a join is a Cartesian product filtered by the join condition.
2. **C** — 7 × 7 = 49.
3. **B** — `LEFT JOIN` keeps unmatched left rows with `NULL`s on the right.
4. **B** — it matches on *all* shared column names implicitly; a later schema change silently alters results.
5. **B** — `NULL = 'shipped'` is unknown, so the outer-added `NULL` rows are filtered out; the `LEFT JOIN` acts like an `INNER JOIN`. Fix by moving the predicate into `ON` or adding `OR o.status IS NULL`.
6. **B** — a `LEFT JOIN` preserving the employee keeps the manager-less CEO with a `NULL` manager.
7. **B** — the strict `<` drops self-pairs and the mirror-image duplicate, keeping each unordered pair once.
8. **B** — `NOT EXISTS` is `NULL`-safe and portable. (D is malformed — an inner join can't produce the `IS NULL` anti-join.)
9. **B** — a `NULL` in the `NOT IN` list makes the predicate never true; use `NOT EXISTS`.
10. **B** — `UNION` de-duplicates (extra sort/hash work); `UNION ALL` keeps everything and is faster.
11. **B** — same column count, compatible types, in order. Result names come from the first query.
12. **C** — West (region 4) has a supplier (WestParts) but no customers. The reverse, `customers EXCEPT suppliers`, is empty.
13. **B** — `EXCEPT` on the key returns only the id; when you need extra columns, use an anti-join against the full table.

</details>

If you scored 11+: take the mini-project and move on. 8–10: re-read the lecture
behind each miss. <8: re-read Lectures 1 and 2 from the top before Week 3.
