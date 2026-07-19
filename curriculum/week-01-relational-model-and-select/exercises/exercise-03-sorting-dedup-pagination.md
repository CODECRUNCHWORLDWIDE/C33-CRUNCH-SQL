# Exercise 3 — Sorting, Dedup & Pagination

**Goal:** Control the *shape* of a result — order it, strip duplicates, and page through it. These three clauses turn "here are some rows" into "here is exactly the report I want."

**Estimated time:** 60 minutes.

## Setup

Connected, 30 rows. New `solutions.sql`. SQLite: `.headers on`, `.mode column`.

## Tasks

### ORDER BY

1. **Single-key sort.** All employees' `first_name` and `salary`, **highest salary first**. *(Top row: Grace, 320000.)*

2. **Ascending.** Same columns, **lowest salary first**. *(Top row: Arjun, 58000. `ASC` is the default — try it with and without the keyword.)*

3. **Multi-key sort.** `department`, `last_name`, `salary`, ordered by **`department` ascending, then `salary` descending within each department**. *(So within Engineering, Ada (240000) comes before Barbara (98000).)*

4. **Sort by an expression.** `first_name` and a computed `tenure_days` (today minus `hire_date`), longest-tenured first. *(Postgres: `CURRENT_DATE - hire_date`. SQLite: `julianday('now') - julianday(hire_date)`. Top row should be Grace, hired 2014-01-06.)*

5. **NULLs in a sort.** `first_name` and `commission_pct` ordered by `commission_pct` **descending**. Note where the `NULL`s land by default. Then force them to the bottom: Postgres `ORDER BY commission_pct DESC NULLS LAST`; SQLite `ORDER BY commission_pct IS NULL, commission_pct DESC`. *(Only the 6 Sales people have a value; the other 24 are `NULL`.)*

### DISTINCT

6. **Distinct values.** The list of **distinct departments**. *(Expected: 8 rows.)*

7. **Distinct combinations.** Distinct **(department, is_remote)** pairs, ordered by department. *(Expected: fewer than 16 — some departments are all-remote or all-office. Count them.)*

8. **Distinct countries.** How many **distinct countries** do employees live in? Return the sorted list. *(Expected: 6 — USA, Canada, UK, Germany, Spain, Brazil, India… count carefully; is that 6 or 7?)*

### LIMIT / OFFSET

9. **Top N.** The **5 highest-paid** employees: `first_name`, `salary`. *(Grace, William, Ada, Don, Alan.)*

10. **Bottom N.** The **3 lowest-paid** employees. *(Arjun, Amelia, Ethan.)*

11. **Pagination — page 2.** Order everyone by `salary` descending, then return **rows 6–10** (the second page of a 5-per-page listing) using `LIMIT` and `OFFSET`. *(Row 6 overall is Margaret, 168000.)*

12. **Newest hires.** The **3 most recently hired** employees: `first_name`, `hire_date`. *(Amelia 2023-06-01, Arjun 2023-03-20, Emma 2023-01-09.)*

13. **A "why is this wrong?" task.** Run `SELECT first_name, salary FROM employees LIMIT 5;` **without** an `ORDER BY`. In a comment, explain why this does **not** reliably return the "first" or "top" 5 anything, and what you'd add to make it meaningful.

## Done when…

- [ ] All 13 queries in `solutions.sql`.
- [ ] Task 3 correctly groups by department **and** sorts within each by descending salary.
- [ ] Task 5 shows you can control `NULL` placement on **both** engines.
- [ ] Task 11 returns rows 6–10, and you can state the `OFFSET` value you used and why.
- [ ] Task 13 has a written explanation about unordered `LIMIT`.

## Stretch

- Combine everything: the **3rd-through-5th highest-paid remote employees**, `first_name` and `salary`, tie-broken by most recent hire. *(One query: `WHERE`, `ORDER BY` two keys, `LIMIT 3 OFFSET 2`.)*
- `SELECT DISTINCT city, country` — how many distinct **city** values are there vs distinct **(city, country)** pairs? Are any city names reused across countries in this dataset?
- Return each **distinct department** alongside… nothing else — then try adding `salary` to the `SELECT DISTINCT` and watch the row count jump. Explain why.

## Submission

Commit `solutions.sql` to your portfolio under `c33-week-01/exercise-03/`.
