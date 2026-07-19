# Week 5 — Exercises

Three guided reps. Do them **in order** — each assumes the mental model the previous one built, and each needs you to actually run SQL, not just read it. Exercises 2 and 3 require **two `psql` sessions open side by side**; split your terminal or open two windows before you start.

| # | Exercise | Skill drilled | Sessions | Time |
|--:|----------|---------------|:--------:|-----:|
| 1 | [exercise-01-transactions-and-savepoints.md](./exercise-01-transactions-and-savepoints.md) | Drive a multi-statement transaction; partial rollback with `SAVEPOINT` / `ROLLBACK TO` | 1 | ~40m |
| 2 | [exercise-02-reproduce-an-anomaly.md](./exercise-02-reproduce-an-anomaly.md) | Reproduce a non-repeatable read, a phantom, and write skew — and prove Postgres won't dirty-read | 2 | ~1h |
| 3 | [exercise-03-cause-and-resolve-a-deadlock.md](./exercise-03-cause-and-resolve-a-deadlock.md) | Deliberately deadlock two sessions, read the error, then fix it with lock ordering | 2 | ~45m |

## Before you start

You need **PostgreSQL 16** running and a database you can throw away. Create one:

```bash
createdb c33_week05        # or: psql -c 'CREATE DATABASE c33_week05;'
psql c33_week05            # opens session A
# in a second terminal:
psql c33_week05            # opens session B
```

In every exercise, a "session" is one `psql` prompt. When a step says **Session A**, type it at the first prompt; **Session B**, the second. The whole point is to watch them interfere in real time, so keep both windows visible.

## How to submit

For each exercise, keep a `solutions.md` with the commands you ran and the output pasted beneath (copy straight from `psql`). Commit them under `c33-week-05/exercise-0N/` in your portfolio. The "Done when…" checklist at the bottom of each file is your self-grade.

## A note on cleanup

Each exercise creates its own tables and drops them at the top with `DROP TABLE IF EXISTS`. If a session gets stuck (a transaction left open holding a lock), the fastest reset is to type `ROLLBACK;` in every open session, or just close and reopen `psql`. A transaction you forgot to close is the #1 reason "the next step just hangs" — check for a dangling `BEGIN` first.
