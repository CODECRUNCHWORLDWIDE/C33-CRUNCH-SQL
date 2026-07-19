# Exercise 2 — Filtering & Pattern Matching

**Goal:** Master the `WHERE` toolkit — boolean logic with parentheses, `IN`, `BETWEEN`, `LIKE`/`ILIKE`, and `IS NULL`. By the end you can translate a precise condition into a precise filter.

**Estimated time:** 60 minutes.

## Setup

Connected, 30 rows in `employees`. New `solutions.sql` under `-- Task N` comments. SQLite users: `.headers on` and `.mode column` for readable output.

## Tasks

1. **AND.** Everyone in Engineering **who is remote**. Return `first_name`, `city`. *(Expected: 5 rows.)*

2. **OR with parentheses.** Everyone in **Sales or Marketing** whose `salary` is **over 100000**. Return `first_name`, `department`, `salary`. *(Think about precedence — you need parentheses. Expected: 6 rows.)*

3. **NOT.** Everyone **not** based in the `'USA'`. Return `first_name`, `country`. *(Expected: 18 rows.)*

4. **IN.** Everyone in Support, HR, **or** Operations, using `IN` (not a chain of `OR`s). *(Expected: 9 rows.)*

5. **NOT IN.** Everyone whose department is **not** Engineering, Sales, or Marketing, using `NOT IN`. *(Expected: 13 rows. `department` is `NOT NULL`, so `NOT IN` is safe here — note *why* that matters.)*

6. **BETWEEN (numbers).** Everyone earning **between 80000 and 120000 inclusive**. Return `first_name`, `salary`, ordered by salary. *(Expected: 12 rows.)*

7. **BETWEEN (dates).** Everyone **hired during 2021** (Jan 1 to Dec 31, 2021). Return `first_name`, `hire_date`. *(Expected: 4 rows.)*

8. **LIKE — starts with.** Everyone whose `last_name` starts with **`S`**. Return `last_name`. *(Expected: 3 rows: Silva, Schmidt, Sharma. On Postgres this is case-sensitive; on SQLite `LIKE` is case-insensitive.)*

9. **LIKE — contains.** Everyone whose `job_title` **contains** `Engineer`. Return `first_name`, `job_title`. *(Expected: 9 rows — the pattern `%Engineer%` also matches "VP Engineering", plus the two plain "Engineer" titles, both "Senior Engineer"s, "Staff", "Junior", and both "Support Engineer"s.)*

10. **LIKE with `_`.** Everyone whose `first_name` has **exactly 3 letters**. Try it two ways: with `LIKE '___'` (three underscores) and with `LENGTH(first_name) = 3`. Confirm they agree. *(Expected: 4 rows — Ada, Don, Ken, Mia.)*

11. **Case-insensitive.** Find the employee whose last name is `turing` **regardless of case**. On Postgres use `ILIKE`; write the portable version too with `LOWER(last_name) = 'turing'`. *(Expected: 1 row.)*

12. **IS NULL.** Everyone with **no email on file**. Return `emp_id`, `first_name`. *(Expected: 2 rows — Emma Schmidt and Benjamin Klein.)*

13. **IS NOT NULL + a domain pattern.** Everyone **whose email ends in `@crunch.io`**. Return `first_name`, `email`. *(Expected: 28 rows — the two NULL emails are correctly excluded, because `NULL LIKE '...'` is `NULL`, not true.)*

14. **The three-valued trap.** Run `SELECT COUNT(*) FROM employees WHERE commission_pct <> 0.10;`. Explain in a comment why the result is **not** 29. *(All the non-Sales rows have `NULL` commission, so `NULL <> 0.10` is unknown and those rows are dropped.)*

## Done when…

- [ ] All 14 queries in `solutions.sql`, row counts matching the "Expected" hints.
- [ ] Task 2 uses parentheses and you can explain what happens *without* them.
- [ ] Task 10's two approaches return the **same** rows.
- [ ] Task 12 uses `IS NULL`, not `= NULL`.
- [ ] Task 14 has a written explanation of the `NULL` behavior.

## Stretch

- Everyone hired in **2018 or later** **and** earning **under 90000** **and** who is **remote**. How many?
- Everyone whose email's **local part** (before `@`) is longer than 12 characters. Mind the `NULL`s.
- Rewrite Task 6 without `BETWEEN`, using `>=` and `<=`. Confirm identical results — then change one bound to make it exclusive and note the row-count change.

## Submission

Commit `solutions.sql` to your portfolio under `c33-week-01/exercise-02/`.
