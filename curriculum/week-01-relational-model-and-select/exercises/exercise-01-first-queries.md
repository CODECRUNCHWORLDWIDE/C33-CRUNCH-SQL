# Exercise 1 — First Queries

**Goal:** Get `SELECT` into your fingers — project columns, compute expressions, alias them, and write your first filters. By the end, typing a basic query feels automatic.

**Estimated time:** 45 minutes.

## Setup

You already ran the seed (see the [week README](../README.md)). Confirm you're connected and the table is there:

```sql
SELECT COUNT(*) FROM employees;   -- must print 30
```

In `psql`, run `\d employees` to see the columns; in SQLite, `.schema employees`. Keep that column list handy.

Create a file `solutions.sql` and put each answer under a `-- Task N` comment.

## Tasks

Write a query for each. Run it, and compare to the "Expected" hint.

1. **Everything.** Select all columns and all rows. *(Expected: 30 rows. This is the one time `SELECT *` is fine — you're exploring.)*

2. **Just three columns.** Select only `first_name`, `last_name`, and `department` for all employees. *(Expected: 30 rows, 3 columns.)*

3. **A full name.** Select a single column that combines first and last name into `"Grace Hopper"` form, aliased as `full_name`. *(Hint: the `||` operator and a `' '` literal.)*

4. **Monthly pay.** Select `first_name`, `salary`, and a computed column `monthly_pay` = salary divided by 12, rounded to 2 decimals. *(Watch integer division — use `/ 12.0` or `ROUND(..., 2)`.)*

5. **Rename in output.** Select `salary` aliased as `annual_salary` and `hire_date` aliased as `"Start Date"` (note the space — you'll need double quotes).

6. **First filter.** Select `first_name`, `last_name`, `salary` for everyone earning **more than 150000**. *(Expected: 8 rows — the CEO, CFO, both VPs, the Marketing Director, plus a Staff and two Senior Engineers. Verify the count.)*

7. **Exact match.** Select all Engineering employees' `first_name` and `job_title`. *(Expected: 7 rows.)*

8. **A date filter.** Select `first_name` and `hire_date` for everyone **hired on or after 2020-01-01**. *(Hint: `hire_date >= '2020-01-01'`. Expected: 13 rows.)*

9. **Not equal.** Select `first_name` and `department` for everyone **not** in Engineering. *(Expected: 23 rows.)*

10. **Expression in the filter.** Select `first_name` and `salary` for everyone whose **monthly** pay exceeds 10000 (i.e., annual salary over 120000). Write the condition on the annual column.

## Expected result (spot checks)

- Task 1 → 30 rows.
- Task 6 → 8 rows (Grace, Ada, Alan, Linus, Margaret, Don, Sofia, William).
- Task 8 → 13 rows.
- Task 9 → 23 rows (30 minus the 7 engineers).

## Done when…

- [ ] `solutions.sql` has all 10 queries, each under a `-- Task N` comment.
- [ ] Task 3 produces a single `full_name` column reading like `Grace Hopper`.
- [ ] Task 4's `monthly_pay` shows 2 decimal places, not an integer.
- [ ] You can explain why Task 5's `"Start Date"` needs **double** quotes, not single.
- [ ] Your row counts match the spot checks above.

## Stretch

- Add a query: `first_name`, `email`, and a column `email_user` = the part of the email before the `@`. *(Postgres: `SPLIT_PART(email,'@',1)`. Both engines: `SUBSTRING(email FROM 1 FOR POSITION('@' IN email) - 1)` — but mind the two `NULL` emails.)*
- Select everyone's `first_name` in ALL CAPS using `UPPER`.

## Submission

Commit `solutions.sql` to your portfolio under `c33-week-01/exercise-01/`.
