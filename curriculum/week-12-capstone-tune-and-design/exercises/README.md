# Week 12 — Exercises

Three guided reps that walk the full tuning loop against **one** deliberately slow, seeded database. Do them in order — each builds on the previous, and together they are a dress rehearsal for the capstone.

1. **[Exercise 1 — Profile a slow database](exercise-01-profile-a-slow-database.md)** — seed a realistic 5M-row database and use `pg_stat_statements` + `EXPLAIN ANALYZE` to find its three worst queries.
2. **[Exercise 2 — Propose and apply fixes](exercise-02-propose-and-apply-fixes.md)** — form a hypothesis for each offender and apply exactly one fix at a time.
3. **[Exercise 3 — Verify with before/after plans](exercise-03-verify-with-before-after-plans.md)** — prove each fix worked with a side-by-side plan and a p95 number.

## Setup you do once (Exercise 1 builds it)

- PostgreSQL 16+ running locally.
- `pg_stat_statements` enabled in `postgresql.conf` (see Exercise 1 for the two lines + restart).
- A scratch database named `crunch_tune` — you will `DROP` and recreate it freely.

## Suggested workflow

- Keep a `report.md` open from Exercise 1 onward. Paste every plan and every number into it as you go — by Exercise 3 you will have most of a tuning report already written.
- Type the SQL yourself. Reading a plan is a skill that only comes from reading many plans.
- After each change, ask out loud: "what did I predict, and did the number move the way I predicted?" If not, revert and rethink.

The goal is not to finish fast. The goal is to make the measure → read → change → verify loop automatic.
