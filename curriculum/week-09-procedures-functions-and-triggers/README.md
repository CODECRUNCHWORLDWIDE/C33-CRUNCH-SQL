# Week 9 — Procedures, Functions & Triggers

> **Goal:** Move logic *into* the database. By the end of this week you can write PL/pgSQL functions with real control flow and error handling, encapsulate queries behind views and materialized views, and wire triggers that keep audit logs and derived columns correct automatically — and you know when *not* to reach for a trigger.

Welcome to the week where the database stops being a passive store you query and starts being a place you can *program*. Weeks 1–8 taught you to ask the database questions. This week you teach it to answer some of them on its own: a `view` that names a gnarly join once and forever, a `function` that runs a multi-step calculation server-side, a `trigger` that fires the instant a row changes so an audit table can never drift out of sync.

This is powerful and it is dangerous. Logic in the database is invisible to application developers who only read the app code — a trigger firing on `INSERT` is "spooky action at a distance" if nobody documented it. So every lecture this week pairs the mechanism with the judgement: *when is this the right tool, and when is it a foot-gun?*

We use **PostgreSQL 16** for everything with server-side logic (SQLite has no stored procedures and only limited triggers — we call out the differences where they matter).

## Learning objectives

By the end of this week you can:

- **Create** a view and a materialized view, explain the difference (virtual vs. stored), and choose between them by read/write ratio and freshness tolerance.
- **Refresh** a materialized view with and without `CONCURRENTLY`, and know what a unique index has to do with it.
- **Write** SQL functions (`LANGUAGE sql`) and PL/pgSQL functions (`LANGUAGE plpgsql`), and say when each is the better choice.
- **Use** PL/pgSQL control flow: variables, `IF/ELSIF`, `CASE`, `LOOP`/`WHILE`/`FOR`, `RETURN`, `RETURN QUERY`, and `RETURNS TABLE`.
- **Handle** errors with `BEGIN ... EXCEPTION WHEN ... END` blocks and `RAISE`.
- **Write** `BEFORE` and `AFTER` triggers at `ROW` and `STATEMENT` level, using `NEW`, `OLD`, and `TG_OP`.
- **Implement** the two most common trigger patterns: an append-only audit log and a maintained derived column.
- **Recognize** the trigger anti-patterns — hidden business logic, cross-table cascades that should be application code, and performance traps.

## Prerequisites

- Weeks 1–8 of C33 (you can write joins, aggregates, CTEs, and window functions without a reference).
- Week 5's transaction model (triggers run *inside* the transaction that fired them — that matters).
- A working `psql` connection to a PostgreSQL 16 database and the course seed dataset loaded.

## Weekly schedule

The schedule below adds up to approximately **28 hours**. Treat it as a target, not a contract.

| Day       | Focus                                       | Lectures | Exercises | Challenges | Quiz/Read | Homework | Mini-Project | Daily Total |
|-----------|---------------------------------------------|---------:|----------:|-----------:|----------:|---------:|-------------:|------------:|
| Monday    | Views + materialized views                  |    2h   |    1.5h   |     0h     |    0.5h   |   1h     |     0h       |    5h       |
| Tuesday   | SQL vs PL/pgSQL functions                   |    2h   |    2h     |     0h     |    0.5h   |   1h     |     0h       |    5.5h     |
| Wednesday | Triggers — mechanics + patterns             |    2h   |    2h     |     1h     |    0.5h   |   1h     |     0h       |    6.5h     |
| Thursday  | Challenges (mview refresh + rule enforce)   |    0h   |    0h     |     2.5h   |    0h     |   1h     |     1h       |    4.5h     |
| Friday    | Mini-project build                          |    0h   |    0h     |     0h     |    0.5h   |   0h     |     3h       |    3.5h     |
| Saturday  | Mini-project polish + quiz                  |    0h   |    0h     |     0h     |    1h     |   1h     |     1h       |    3h       |
| **Total** |                                             | **6h**  | **5.5h**  | **3.5h**   | **3h**    | **5h**   | **5h**       | **28h**     |

## How to navigate this week

Work top to bottom. Each lecture has exercises that drill it; do them the same day.

| # | File | What's inside | Time |
|---|------|---------------|------|
| 1 | [lecture-notes/01-views-and-materialized-views.md](./lecture-notes/01-views-and-materialized-views.md) | Views vs. materialized views: creation, `REFRESH`, `CONCURRENTLY`, when to use each | ~2h |
| 2 | [lecture-notes/02-functions-sql-and-plpgsql.md](./lecture-notes/02-functions-sql-and-plpgsql.md) | SQL functions vs PL/pgSQL: variables, control flow, `RETURNS TABLE`, exception handling | ~2h |
| 3 | [lecture-notes/03-triggers.md](./lecture-notes/03-triggers.md) | `BEFORE`/`AFTER`, `ROW`/`STATEMENT`, audit-log + derived-column patterns, when NOT to use triggers | ~2h |
| 4 | [exercises/README.md](./exercises/README.md) | Index of the three guided exercises | — |
| 5 | [exercises/exercise-01-view-and-sql-function.md](./exercises/exercise-01-view-and-sql-function.md) | Build a view + a SQL function over the seed schema | ~40m |
| 6 | [exercises/exercise-02-plpgsql-function-control-flow.md](./exercises/exercise-02-plpgsql-function-control-flow.md) | A PL/pgSQL function with branching + error handling | ~50m |
| 7 | [exercises/exercise-03-audit-trigger.md](./exercises/exercise-03-audit-trigger.md) | An `AFTER` row trigger that writes an audit log | ~50m |
| 8 | [challenges/README.md](./challenges/README.md) | Index of the two open-ended challenges | — |
| 9 | [challenges/challenge-01-materialized-view-refresh-strategy.md](./challenges/challenge-01-materialized-view-refresh-strategy.md) | Design a refresh strategy for a reporting mview | ~75m |
| 10 | [challenges/challenge-02-enforce-business-rule-with-trigger.md](./challenges/challenge-02-enforce-business-rule-with-trigger.md) | Enforce a cross-row business rule a `CHECK` can't express | ~75m |
| 11 | [mini-project/README.md](./mini-project/README.md) | Ship an audited table with triggers + functions | ~5h |
| 12 | [homework.md](./homework.md) | Six practice tasks reinforcing the week | ~5h |
| 13 | [quiz.md](./quiz.md) | 13 self-check questions + answer key | ~1h |
| 14 | [resources.md](./resources.md) | Official docs + curated further reading | — |

## By the end of this week you can…

- Name a complex query once with a **view** and reuse it everywhere.
- Trade freshness for speed deliberately with a **materialized view** and a refresh plan.
- Push a multi-step calculation into a **PL/pgSQL function** that branches, loops, and handles errors.
- Guarantee an **audit trail** and keep **derived columns** correct with triggers that can't be bypassed.
- Argue *both sides* of "should this be a trigger?" and pick correctly.

## Up next

[Week 10 — Security & Data Integrity](../week-10-security-and-data-integrity/) — once your audited table passes its rubric. Functions and triggers set up roles and `SECURITY DEFINER`, which Week 10 leans on hard.

---

*Part of C33 · Crunch SQL · GPL-3.0. Found an error? Open an issue or PR.*
