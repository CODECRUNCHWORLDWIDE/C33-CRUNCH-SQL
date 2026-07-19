# Week 1 — Quiz

Fourteen questions. Lectures closed. Aim for 12/14 before starting Week 2. A mix of multiple-choice and short "what does this return?" — the answer key at the bottom explains the *why*, not just the letter.

---

**Q1.** In the relational model, the everyday word for a *tuple* is:

- A) A column
- B) A table
- C) A row
- D) A key

---

**Q2.** Which statement about a **primary key** is true?

- A) It may be `NULL` as long as it's unique.
- B) It is unique **and** never `NULL`.
- C) It must reference another table.
- D) A table can have several primary keys.

---

**Q3.** What links employee 3's row to employee 2's row in our `employees` table?

- A) A `DISTINCT` clause
- B) The `emp_id` primary key alone
- C) The `manager_id` foreign key
- D) An `ORDER BY`

---

**Q4.** Why is `SELECT * FROM employees LIMIT 5;` (with no `ORDER BY`) a code smell?

- A) `LIMIT` is not valid without `ORDER BY`.
- B) A table is an unordered set, so "the first 5" is undefined and can change between runs.
- C) `*` is illegal in PostgreSQL.
- D) It always returns the 5 lowest `emp_id`s.

---

**Q5.** Which of these correctly tests for a missing email?

- A) `WHERE email = NULL`
- B) `WHERE email == NULL`
- C) `WHERE email IS NULL`
- D) `WHERE email = ''`

---

**Q6.** `WHERE salary BETWEEN 80000 AND 120000` includes:

- A) 80000 but not 120000
- B) Neither endpoint
- C) Both 80000 and 120000
- D) 120000 but not 80000

---

**Q7.** On **PostgreSQL**, which matches `Smith`, `smith`, and `SMITH`?

- A) `LIKE 'smith'`
- B) `ILIKE 'smith'`
- C) `= 'smith'`
- D) `LIKE 'S%'` only

---

**Q8.** `SELECT 5 / 2;` in PostgreSQL returns:

- A) `2.5`
- B) `2`
- C) `3`
- D) An error

---

**Q9.** Why does `WHERE emp_id NOT IN (1, 2, NULL)` return **zero** rows?

- A) `NOT IN` is invalid syntax.
- B) It expands to `... AND emp_id <> NULL`, which is `unknown`, so no row is ever `true`.
- C) `NULL` equals every integer.
- D) `emp_id` can't be compared with `IN`.

---

**Q10.** `SELECT DISTINCT department, is_remote FROM employees;` returns:

- A) One row per department
- B) One row per distinct **(department, is_remote)** pair
- C) One row per employee
- D) Only the `department` column, deduplicated

---

**Q11.** Which clause can reference an alias defined in the `SELECT` list?

- A) `WHERE`
- B) `FROM`
- C) `ORDER BY`
- D) None can

---

**Q12.** `AVG(commission_pct)` over all 30 employees computes the average of:

- A) All 30 rows, treating `NULL` as 0
- B) Only the rows where `commission_pct` is not `NULL` (the 6 Sales staff)
- C) Zero — because some values are `NULL`
- D) The `emp_id` column by mistake

---

**Q13.** You want the second page of a 5-per-page listing sorted by salary descending. Which is correct?

- A) `ORDER BY salary DESC LIMIT 5`
- B) `ORDER BY salary DESC LIMIT 5 OFFSET 5`
- C) `ORDER BY salary DESC OFFSET 5`
- D) `LIMIT 5 OFFSET 5` (no `ORDER BY`)

---

**Q14.** `COALESCE(commission_pct, 0)` returns:

- A) Always `0`
- B) `NULL` when the commission is `0`
- C) The commission if present, otherwise `0`
- D) The commission rounded to zero decimals

---

## Answer key

<details>
<summary>Reveal after attempting</summary>

1. **C** — a tuple is a row. (Attribute = column, relation = table.)
2. **B** — a primary key is unique **and** not null. That's the whole definition. A *unique constraint* may allow one `NULL`; a *foreign key* references another key.
3. **C** — the `manager_id` foreign key. Employee 3 (Alan) has `manager_id = 2` (Ada). Relationships are represented by matching a value to a key elsewhere.
4. **B** — a table is an unordered set of rows, so `LIMIT` without `ORDER BY` returns an arbitrary, non-deterministic slice. `LIMIT` *is* legal without `ORDER BY` — it's just meaningless.
5. **C** — `IS NULL`. `= NULL` and `== NULL` are never true; `= ''` tests for empty string, which is a *different* thing from `NULL`.
6. **C** — `BETWEEN` is inclusive of **both** endpoints (`>= 80000 AND <= 120000`).
7. **B** — `ILIKE` is PostgreSQL's case-insensitive match. (On SQLite, plain `LIKE` would already do this.)
8. **B** — `2`. Integer ÷ integer truncates in Postgres. Use `5 / 2.0` for `2.5`.
9. **B** — the `NULL` in the list produces `emp_id <> NULL` → `unknown`, and `anything AND unknown` can never be `true`, so every row is dropped.
10. **B** — `DISTINCT` applies to the whole projected row, so you get distinct **pairs**, not distinct departments.
11. **C** — `ORDER BY` runs after `SELECT`, so it can see aliases. `WHERE` runs before `SELECT` and cannot.
12. **B** — aggregates ignore `NULL`. `AVG` divides the sum of non-null commissions by the *count of non-null* commissions — the 6 Sales staff, not 30.
13. **B** — `ORDER BY salary DESC LIMIT 5 OFFSET 5` skips the top 5 and returns ranks 6–10. Without `ORDER BY` (D) the "page" is meaningless.
14. **C** — `COALESCE` returns its first non-`NULL` argument: the commission if it exists, else `0`. Note it does **not** turn a real `0` into `NULL`.

</details>

**Scoring:** 12+ → start Week 2. 9–11 → re-read the lecture sections behind your misses. <9 → re-read all three lectures from the top; the concepts compound fast next week.
