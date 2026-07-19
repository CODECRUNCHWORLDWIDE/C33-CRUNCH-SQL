# Week 3 — Aggregation, Subqueries & CTEs

> *Weeks 1–2 kept every row visible. This week you learn to **summarise** — collapse thousands of rows into the one number a business actually asked for — then to **nest** queries inside queries, and finally to **name** your intermediate results so a ten-join analytics query reads top to bottom instead of inside-out.*

Welcome to Week 3 of **C33 · Crunch SQL**. You already query one table and join many. Now you turn raw rows into answers: `GROUP BY` with `HAVING`, the aggregate functions and `FILTER`, subtotals with `ROLLUP`/`CUBE`, every flavour of subquery (scalar, `IN`, `ANY`/`ALL`, `EXISTS`, and the correlated ones that re-run per row), and CTEs — including a **recursive** CTE that walks a category tree or an org chart to any depth. By Sunday you can build a layered analytics query as a clean pipeline of named stages, which is exactly what the mini-project asks for.

Everything this week runs against one small store database, **`crunch_shop`**, that you load once from `exercises/README.md`. It's tiny on purpose — you can hand-check every answer.

## Learning objectives

By the end of this week, you will be able to:

- **Aggregate** with `COUNT`/`SUM`/`AVG`/`MIN`/`MAX`, and know exactly how each treats NULLs (and how `COUNT(*)` differs from `COUNT(col)`).
- **Group and filter** with `GROUP BY` and `HAVING`, and always put a condition in the right one of `WHERE` vs `HAVING`.
- **Slice within a group** using `FILTER (WHERE …)` instead of clumsy `CASE` sums.
- **Produce subtotals** with `GROUPING SETS`, `ROLLUP`, and `CUBE` — and know these are Postgres-only among our engines.
- **Write every subquery shape** — scalar, `IN`, `ANY`/`ALL`, `EXISTS` — and dodge the `NOT IN` NULL trap with `NOT EXISTS`.
- **Recognise a correlated subquery** (the one that references the outer row) and reason about when it runs once vs per-row.
- **Refactor into CTEs**: lift subqueries into named `WITH` blocks and chain them into a readable pipeline.
- **Walk a hierarchy** with a recursive CTE, carrying a depth counter and always shipping a loop guard.

## Prerequisites

- **C33 Weeks 1–2** complete: you can `SELECT`/filter/sort one table and join several with every join type plus `UNION`/`INTERSECT`/`EXCEPT`.
- PostgreSQL 16 **or** SQLite installed and a client (`psql` / `sqlite3`) you can run. Both engines work all week; engine-specific gaps (grouping sets, `LATERAL`, `CYCLE`) are called out where they matter.

## How to navigate this week

Numbered in teaching order. Do the lectures, then the exercises, then challenges, then the mini-project; homework and quiz reinforce throughout.

| # | File | What's inside | Time |
|--:|------|---------------|-----:|
| 1 | [lecture-notes/01-aggregation-group-by-having.md](./lecture-notes/01-aggregation-group-by-having.md) | Aggregates, `GROUP BY`, `HAVING`, `FILTER`, `ROLLUP`/`CUBE` | ~2h |
| 2 | [lecture-notes/02-subqueries.md](./lecture-notes/02-subqueries.md) | Scalar, `IN`, `ANY`/`ALL`, `EXISTS`, and correlated subqueries | ~2h |
| 3 | [lecture-notes/03-ctes-and-recursion.md](./lecture-notes/03-ctes-and-recursion.md) | `WITH`, chaining, the optimisation fence, recursive CTEs | ~2h |
| 4 | [exercises/README.md](./exercises/README.md) | Seed database setup + exercise index | 5 min |
| 5 | [exercises/exercise-01-group-and-aggregate.md](./exercises/exercise-01-group-and-aggregate.md) | Eight grouping/aggregation questions | ~45m |
| 6 | [exercises/exercise-02-subqueries.md](./exercises/exercise-02-subqueries.md) | All four subquery shapes, incl. correlated + `EXISTS` | ~50m |
| 7 | [exercises/exercise-03-ctes.md](./exercises/exercise-03-ctes.md) | Chained CTEs + two recursive CTEs | ~55m |
| 8 | [challenges/challenge-01-layered-analytics.md](./challenges/challenge-01-layered-analytics.md) | A customer scorecard with nasty edges | ~75m |
| 9 | [challenges/challenge-02-top-n-per-group.md](./challenges/challenge-02-top-n-per-group.md) | Top-2 products per category, no window functions | ~75m |
| 10 | [mini-project/README.md](./mini-project/README.md) | One layered country-performance report, built from CTEs | 4–5h |
| 11 | [homework.md](./homework.md) | Six reinforcement problems | ~5h |
| 12 | [quiz.md](./quiz.md) | 15 self-check questions with answer key | ~30m |
| — | [resources.md](./resources.md) | Curated official docs + practice sites | — |

## Weekly schedule

Roughly **28 hours**, matching the course's full-time pace. Treat it as a target.

| Day | Focus | Lectures | Exercises | Challenges | Homework | Mini-Project | Quiz | Total |
|-----|-------|---------:|----------:|-----------:|---------:|-------------:|-----:|------:|
| Mon | Aggregation + GROUP BY/HAVING | 2h | 1h | 0h | 1h | 0h | 0h | 4h |
| Tue | FILTER, ROLLUP; subqueries begin | 2h | 1h | 0h | 1.5h | 0h | 0h | 4.5h |
| Wed | Subqueries: EXISTS + correlated | 2h | 1h | 1.25h | 1h | 0h | 0h | 5.25h |
| Thu | CTEs + recursion | 2h | 1h | 1.25h | 1h | 0h | 0h | 5.25h |
| Fri | Challenges | 0h | 0h | 0h | 0.5h | 2h | 0h | 2.5h |
| Sat | Mini-project | 0h | 0h | 0h | 0h | 3h | 0h | 3h |
| Sun | Mini-project wrap + quiz | 0h | 0h | 0h | 0h | 2h | 0.5h | 2.5h |
| **Total** | | **10h** | **5h** | **3.75h** | **6.5h** | **7h** | **0.5h** | **~28h** |

## By the end of this week you can…

- Answer a real business question by collapsing thousands of rows into the right number, grouped and filtered correctly.
- Choose `WHERE` vs `HAVING`, `IN` vs `EXISTS`, and `NOT EXISTS` over `NOT IN` — for the right reasons.
- Spot a correlated subquery and know whether it's cheap or a trap.
- Decompose a gnarly analytics query into a legible chain of named CTEs.
- Walk a category tree or org chart to any depth with a recursive CTE — safely guarded against runaway loops.

## Up next

[Week 4 — Data modeling & normalization](../week-04-data-modeling-and-normalization/): design the schema those queries deserve — keys, constraints, and 1NF → BCNF.

---

*If you find errors, please open an issue or PR.*
