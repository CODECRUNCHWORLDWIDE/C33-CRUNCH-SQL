# Week 1 — Homework

Five problems, ~5 hours total, spread across the week. These reinforce the lectures with a mix of hands-on queries, a written explanation, and a build-your-own-dataset task. Commit each.

All queries run against the `employees` seed from the [README](./README.md) unless a problem says otherwise.

---

## Problem 1 — Set up both engines (45 min)

Get **both** PostgreSQL and SQLite working, so you can feel the difference firsthand.

1. Install PostgreSQL 16+ and SQLite 3.35+ (see [`resources.md`](./resources.md)).
2. Create the `crunch` database in Postgres and `crunch.db` in SQLite, and load the seed into **both**.
3. Run `SELECT COUNT(*) FROM employees;` in each — both must return `30`.

**Deliver** `setup.md` with: the version of each engine (`SELECT version();` in Postgres, `SELECT sqlite_version();` in SQLite), and one sentence on a difference you noticed connecting to each.

---

## Problem 2 — Twenty warm-up queries (75 min)

Write and run each. Put them in `warmups.sql` with a `-- N` comment and the answer count beneath each.

1. All employees in Marketing.
2. Employees hired before 2016.
3. The three highest salaries (values only, `DISTINCT`).
4. Everyone whose first name ends in "a".
5. Employees in Miami **or** London.
6. Everyone earning 100,000 or less, cheapest first.
7. Distinct `(country, city)` pairs, sorted.
8. Everyone whose `job_title` is exactly `'Engineer'` (not "Senior Engineer").
9. Employees with a commission rate of 0.12 or higher.
10. Everyone **not** remote.
11. The 4th-through-6th highest earners (use `LIMIT`/`OFFSET`).
12. Employees whose email is missing.
13. Everyone hired in the first quarter (Jan–Mar) of any year. *(Hint: `EXTRACT(MONTH …)` in Postgres, `strftime('%m', …)` in SQLite.)*
14. Full name + department, sorted by department then last name.
15. Distinct departments that have at least one remote worker. *(You can do this with `DISTINCT` and a `WHERE` this week.)*
16. Everyone whose salary, rounded to the nearest 10,000, equals 120,000.
17. Employees whose last name is longer than 7 characters.
18. Everyone in Engineering **or** Finance earning over 150,000.
19. The oldest employee by `birth_date` (earliest date).
20. Everyone whose `manager_id` is `NULL` (should be just the CEO).

---

## Problem 3 — Explain the NULL (45 min)

In `null-writeup.md`, answer in prose (no more than 400 words total):

1. Run `SELECT COUNT(*) FROM employees WHERE commission_pct > 0;` and `SELECT COUNT(*) FROM employees WHERE commission_pct <= 0;`. Add the two. Why don't they sum to 30?
2. Explain in your own words the difference between `NULL`, `0`, and `''`.
3. Give a real-world example (not from this course) of a column where `NULL` means "not applicable" and one where it means "unknown, but exists."
4. Write the correct predicate for "everyone whose commission is missing OR zero" and explain each part.

---

## Problem 4 — Case-insensitive search across engines (45 min)

The same task, two engines, three ways. In `search.sql`:

1. **Postgres:** find everyone whose last name starts with "s", case-insensitively, using `ILIKE`.
2. **Both engines:** the same result using `LOWER(last_name) LIKE 's%'`.
3. **SQLite:** the same result using plain `LIKE 's%'` (and note *why* it already works there).

**Deliver** the three queries plus a two-sentence note: when would you prefer `LOWER(...) LIKE` over `ILIKE`, given what you know about portability?

---

## Problem 5 — Extend the dataset (60 min)

Make the data your own and query it.

1. Write `INSERT` statements to add **5 new employees** of your invention. Include at least: one with a `NULL` email, one in a new city, one in Sales (with a commission), and one who is the newest hire.
2. Load them (you now have 35 rows).
3. Write 5 queries that only make sense *because* of your new rows (e.g., "the newest hire is now…", "employees in the new city").
4. Then **undo** it cleanly: `DELETE FROM employees WHERE emp_id >= 31;` and confirm you're back to 30.

**Deliver** `extend.sql` (the inserts, the 5 queries with outputs, and the cleanup) plus one sentence on what surprised you about how your new rows changed earlier answers.

---

## Time budget

| Problem | Time |
|--------:|----:|
| 1 | 45 min |
| 2 | 75 min |
| 3 | 45 min |
| 4 | 45 min |
| 5 | 60 min |
| **Total** | **~4.5 h** |

After homework, take the [quiz](./quiz.md) and ship the [mini-project](./mini-project/README.md).
