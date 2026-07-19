# Week 2 — Joins & Set Operations

> *One table answers one question. The interesting questions live between tables. This week you learn to combine them — every join the relational model offers, plus the set operators that stack results on top of each other.*

Welcome to Week 2 of **C33 · Crunch SQL**. Week 1 made you fluent against a *single* table — filter, sort, `DISTINCT`, `LIMIT`. Real schemas spread their data across many tables on purpose (that is what normalization buys you, and you'll see why in Week 4). To answer anything useful you have to put those tables back together at query time. That reassembly is the **join**, and it is the single most important operation in SQL.

By the end of the week you will reach for `INNER`, `LEFT`, `RIGHT`, and `FULL` joins without hesitation, know exactly which rows each one keeps and drops, handle the `NULL`s that outer joins manufacture, express "rows in A with no match in B" two different ways (the **anti-join**), and stack query results with `UNION`, `INTERSECT`, and `EXCEPT`. You'll finish by reconstructing a business report that lives across **five** joined tables.

## Learning objectives

By the end of this week, you will be able to:

- **Explain** the join model as a filtered Cartesian product, and why the `ON` condition — not the join keyword — decides what matches.
- **Write** all four join types (`INNER`, `LEFT`, `RIGHT`, `FULL OUTER`) and predict their row counts before running them.
- **Choose** between `ON` and `USING`, and know what `NATURAL JOIN` does and why professionals avoid it.
- **Handle** the `NULL`s outer joins produce — in the `SELECT` list, in `WHERE`, and with `COALESCE`.
- **Build** self-joins (a table joined to itself), `CROSS JOIN`s, and multi-table joins of three or more tables.
- **Find missing rows** with an anti-join, written both as `LEFT JOIN … WHERE … IS NULL` and as `NOT EXISTS`.
- **Combine** result sets with `UNION`, `UNION ALL`, `INTERSECT`, and `EXCEPT`, and decide when a set operator beats a join.
- **Read and order** a multi-table query for correctness and for a human.

## Prerequisites

- **C33 Week 1** complete — you can write `SELECT … FROM … WHERE … ORDER BY … LIMIT` against one table.
- A working **PostgreSQL 16** (primary) *or* **SQLite 3.39+** (zero-setup). Setup and the seed dataset live in [`exercises/README.md`](./exercises/README.md); load it once before Tuesday.
- Comfort reading a small entity-relationship sketch. We use one all week — the `crunch_shop` schema (customers, orders, products, and the tables that connect them).

> **SQLite note:** `RIGHT JOIN` and `FULL OUTER JOIN` require **SQLite 3.39** (June 2022) or newer. Check with `sqlite3 --version`. Everything else works on any recent SQLite, and all of it works on PostgreSQL 16.

## Topics covered

- The relational join as `CROSS JOIN` + `WHERE`, and why every join is "just" a filtered product
- `INNER JOIN` — the intersection, the 90% case
- `LEFT` / `RIGHT` / `FULL OUTER JOIN` — keeping unmatched rows on one or both sides
- `ON` vs `USING` vs `NATURAL JOIN` — three ways to state the match condition
- Self-joins — hierarchies (employee → manager) and same-table comparisons
- `CROSS JOIN` — when you actually want the full product (calendars, grids, densification)
- Multi-table joins — chaining three, four, five tables and reading the result
- Anti-joins — `NOT EXISTS` vs `LEFT JOIN … IS NULL`, and semi-joins for contrast
- Set operations — `UNION`, `UNION ALL`, `INTERSECT`, `EXCEPT`, column/type rules, `ALL` semantics
- Joins-vs-sets: combining *columns* (join) vs stacking *rows* (set)

## Weekly schedule

The schedule below adds up to approximately **28 hours** — the C33 full-time pace. Treat it as a target, not a contract.

| Day       | Focus                                       | Lectures | Exercises | Challenges | Quiz/Read | Homework | Mini-Project | Daily Total |
|-----------|---------------------------------------------|---------:|----------:|-----------:|----------:|---------:|-------------:|------------:|
| Monday    | Load the seed DB; inner + outer joins       |    2h    |    2h     |     0h     |    0.5h   |   1h     |     0h       |    5.5h     |
| Tuesday   | ON vs USING; NULL handling in outer joins   |    0h    |    2h     |     1.5h   |    0.5h   |   1h     |     0h       |    5h       |
| Wednesday | Self / cross / multi-table / anti-joins     |    2h    |    2h     |     1h     |    0.5h   |   1h     |     0h       |    6h       |
| Thursday  | Set operations; joins-vs-sets               |    2h    |    1.5h   |     1h     |    0.5h   |   1h     |     0h       |    6h       |
| Friday    | Mini-project — reconstruct the report       |    0h    |    0h     |     0h     |    0h     |   0.5h   |     3h       |    3.5h     |
| Saturday  | Quiz + review                               |    0h    |    0h     |     0h     |    1h     |   0h     |     1h       |    2h       |
| **Total** |                                             | **6h**   | **9.5h**  | **4.5h**   | **3h**    | **5.5h** | **5h**       | **28h**     |

## How to navigate this week

Work top to bottom — each piece assumes the ones above it.

| # | File | What's inside | Time |
|--:|------|---------------|-----:|
| 1 | [README.md](./README.md) | This overview | 15m |
| 2 | [exercises/README.md](./exercises/README.md) | **Load the `crunch_shop` seed DB here first** + exercise index | 30m |
| 3 | [lecture-notes/01-inner-and-outer-joins.md](./lecture-notes/01-inner-and-outer-joins.md) | The join model; `INNER`/`LEFT`/`RIGHT`/`FULL`; `ON` vs `USING` | 2h |
| 4 | [lecture-notes/02-self-cross-multi-and-anti-joins.md](./lecture-notes/02-self-cross-multi-and-anti-joins.md) | Self-joins, cross joins, multi-table joins, anti-joins, join order | 2h |
| 5 | [lecture-notes/03-set-operations.md](./lecture-notes/03-set-operations.md) | `UNION`/`UNION ALL`/`INTERSECT`/`EXCEPT`; joins vs sets | 2h |
| 6 | [exercises/exercise-01-two-table-joins.md](./exercises/exercise-01-two-table-joins.md) | Guided two-table inner joins | 40m |
| 7 | [exercises/exercise-02-outer-joins-and-nulls.md](./exercises/exercise-02-outer-joins-and-nulls.md) | Outer joins + `NULL` handling | 45m |
| 8 | [exercises/exercise-03-self-joins-and-set-ops.md](./exercises/exercise-03-self-joins-and-set-ops.md) | Self-join + set operations | 45m |
| 9 | [challenges/README.md](./challenges/README.md) | Stretch challenges index | 10m |
| 10 | [challenges/challenge-01-reconstruct-a-report.md](./challenges/challenge-01-reconstruct-a-report.md) | Rebuild a multi-table sales report | 75m |
| 11 | [challenges/challenge-02-find-the-missing-matches.md](./challenges/challenge-02-find-the-missing-matches.md) | Anti-join hunt: rows with no partner | 60m |
| 12 | [mini-project/README.md](./mini-project/README.md) | Reconstruct a report across 5 joined tables | 3–5h |
| 13 | [homework.md](./homework.md) | Six practice problems (~5.5h) | 5.5h |
| 14 | [quiz.md](./quiz.md) | 13 self-check questions + answer key | 45m |
| 15 | [resources.md](./resources.md) | Curated docs + join visualizers | — |

## By the end of this week you can…

- Read a two-table `JOIN` and say, out loud, which rows survive and which are dropped.
- Pick the right join type for a question phrased in English ("customers who never ordered" → anti-join; "all customers, with orders where they exist" → `LEFT JOIN`).
- Join five tables into one report and trust the row count.
- Express set membership questions (`in A but not B`, `in both`) with `EXCEPT` / `INTERSECT` *or* with a join, and know which is clearer.
- Debug the two classic join bugs: the accidental Cartesian explosion, and the `LEFT JOIN` silently turned into an `INNER JOIN` by a `WHERE` on the right table.

## Up next

[Week 3 — Aggregation, subqueries & CTEs](../week-03-aggregation-subqueries-and-ctes/) — once every join type is muscle memory, you'll group the joined rows and summarize them.

---

*Part of the Code Crunch Worldwide open curriculum · GPL-3.0 · If you find an error, open an issue or PR.*
