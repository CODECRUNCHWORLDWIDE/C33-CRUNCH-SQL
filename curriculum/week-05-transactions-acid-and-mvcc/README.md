# Week 5 — Transactions, ACID & MVCC

> **Goal:** By the end of this week you can *choose the right isolation level for a workload and avoid the concurrency anomalies each level allows* — and prove, in two live `psql` sessions, exactly which anomaly appears and which one you traded it for.

Weeks 1–4 taught you to query and model data *as if you were the only user*. You are never the only user. The moment two sessions touch the same rows at the same time, a whole new class of bug appears — bugs that never show up in a single-user test, that pass code review, and that corrupt money in production. This is the week you learn to reason about concurrency instead of hoping.

A transaction is the unit of "all or nothing." **ACID** is the four guarantees a database makes about that unit. **Isolation levels** are the dial that trades correctness for concurrency — turn it up and anomalies disappear but contention rises; turn it down and you go faster but must defend yourself. **MVCC** is *how* PostgreSQL actually delivers those guarantees without making readers and writers block each other. Understand all four and you stop being surprised by production.

We use **PostgreSQL 16** as the primary engine because its MVCC implementation is the clearest teaching model in open source, and **SQLite** as a contrast — a database that gives you serializability by refusing to let two writers in at once. Seeing both makes the trade-off obvious.

## By the end of this week you can…

- **Explain** ACID precisely — not as a slogan but as four separate promises, and name what breaks when each one is violated.
- **Drive** transactions by hand: `BEGIN` / `COMMIT` / `ROLLBACK`, and use `SAVEPOINT` / `ROLLBACK TO` to undo part of a transaction without losing the rest.
- **Name and reproduce** the four anomalies — dirty read, non-repeatable read, phantom, and serialization anomaly (write skew) — and say which isolation level allows each.
- **Choose** an isolation level (`READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`) for a given scenario and justify it against latency, contention, and correctness.
- **Read** PostgreSQL's MVCC internals: snapshots, the `xmin`/`xmax` system columns, tuple visibility, and why `VACUUM` exists at all.
- **Take locks on purpose** — `SELECT … FOR UPDATE`, row vs. table locks — read `pg_locks`, cause a **deadlock**, understand how Postgres detects and breaks it, and write a **retry loop** that survives serialization failures.
- **Find and fix a lost update** — the single most common concurrency bug in real codebases.

## Prerequisites

- **C33 Weeks 1–4** complete: you can `SELECT`, `JOIN`, aggregate, and you understand keys and constraints.
- **PostgreSQL 16** installed locally and you can open a `psql` prompt. Install notes in [`resources.md`](./resources.md).
- **SQLite 3** (ships on macOS and most Linux; `sqlite3` on the command line).
- A terminal where you can open **two `psql` sessions side by side** — this week half the learning is watching Session A and Session B interfere. Split your terminal or open two windows.

## How to navigate this week

Work top to bottom. The lectures build the mental model; the exercises make you *do* it in two sessions; the challenges and mini-project make you decide and defend.

| # | File | What's inside | Time |
|--:|------|---------------|-----:|
| 1 | [lecture-notes/01-transactions-and-acid.md](./lecture-notes/01-transactions-and-acid.md) | What a transaction is, ACID as four promises, `BEGIN`/`COMMIT`/`ROLLBACK`, savepoints, autocommit | ~2h |
| 2 | [lecture-notes/02-isolation-levels-and-anomalies.md](./lecture-notes/02-isolation-levels-and-anomalies.md) | The four isolation levels, the four anomalies, which level allows which, with reproductions | ~2h |
| 3 | [lecture-notes/03-mvcc-locking-and-deadlocks.md](./lecture-notes/03-mvcc-locking-and-deadlocks.md) | Postgres MVCC (snapshots, `xmin`/`xmax`, vacuum), row/table locks, `FOR UPDATE`, deadlocks | ~2.5h |
| 4 | [exercises/exercise-01-transactions-and-savepoints.md](./exercises/exercise-01-transactions-and-savepoints.md) | Drive a multi-statement transaction; partial rollback with savepoints | ~40m |
| 5 | [exercises/exercise-02-reproduce-an-anomaly.md](./exercises/exercise-02-reproduce-an-anomaly.md) | Two `psql` sessions: reproduce one anomaly at each isolation level | ~1h |
| 6 | [exercises/exercise-03-cause-and-resolve-a-deadlock.md](./exercises/exercise-03-cause-and-resolve-a-deadlock.md) | Deliberately deadlock two sessions, read the error, then fix the ordering | ~45m |
| 7 | [challenges/challenge-01-pick-the-isolation-level.md](./challenges/challenge-01-pick-the-isolation-level.md) | Four scenarios — pick a level for each and justify the trade-off | ~1h |
| 8 | [challenges/challenge-02-fix-a-lost-update.md](./challenges/challenge-02-fix-a-lost-update.md) | A buggy read-modify-write loses money; fix it three different ways | ~1h |
| 9 | [mini-project/README.md](./mini-project/README.md) | Reproduce *and* fix a concurrency anomaly, documented with a two-session timeline | ~3h |
| — | [homework.md](./homework.md) | Six practice tasks reinforcing the week | ~4h |
| — | [quiz.md](./quiz.md) | 14 self-check questions + answer key | ~40m |
| — | [resources.md](./resources.md) | Official docs + the papers and posts worth your time | — |

Total is roughly **17–18 hours** of directed work. Treat it as a target, not a race — concurrency is a topic you understand by *watching two sessions interfere*, so give the exercises real time.

## The one habit to build this week

Whenever you write code that reads a value, does arithmetic on it, and writes it back — **stop and ask "what if another session runs this same code between my read and my write?"** That single question, asked reflexively, prevents the majority of concurrency bugs you will ever cause. Everything below is the vocabulary and the tools to answer it.

## Up next

[Week 6 — Indexing deep-dive](../week-06-indexing-deep-dive/) — once you can make queries *correct* under concurrency, Week 6 makes them *fast*.

---

*Part of C33 · Crunch SQL · GPL-3.0. Found an error? Open an issue or PR.*
