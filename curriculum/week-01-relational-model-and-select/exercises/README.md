# Week 1 — Exercises

Three guided exercises, ~45–60 min each. **Type every query yourself** — don't copy-paste. Reading SQL and writing SQL are different skills, and only writing sticks.

1. **[Exercise 1 — First queries](exercise-01-first-queries.md)** — projection, aliases, expressions, and your first `WHERE`s.
2. **[Exercise 2 — Filtering & pattern matching](exercise-02-filtering-and-pattern-matching.md)** — `AND`/`OR`/`NOT`, `IN`, `BETWEEN`, `LIKE`/`ILIKE`, `IS NULL`.
3. **[Exercise 3 — Sorting, dedup & pagination](exercise-03-sorting-dedup-pagination.md)** — `ORDER BY`, `DISTINCT`, `LIMIT`/`OFFSET`.

## Before you start

- You've completed all three lectures.
- You ran the seed from the [week README](../README.md) and `SELECT COUNT(*) FROM employees;` returns **30**.
- You have a shell open: `psql crunch` (Postgres) or `sqlite3 crunch.db` (SQLite).

## Suggested workflow

- Open the exercise file beside your terminal.
- For each task, **write your query before running it**, then run it and check the output against the "Expected" note.
- If a result surprises you, stop and figure out *why* before moving on — that confusion is the lesson.
- Save your answers in a `solutions.sql` file (one query per task, with the task number in a `-- comment`). You'll commit it.

## Note on the two engines

Every task works on both PostgreSQL and SQLite. Where the two differ (case-insensitive matching, date functions), the task says so and gives both spellings. If you're on SQLite, run `.headers on` and `.mode column` first so output is readable.
