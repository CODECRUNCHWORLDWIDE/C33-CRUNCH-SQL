# Week 6 — Indexing Deep-Dive

> *An index is a deal you make with the database: pay on every write, and on disk, to make some reads cheap. This week you learn which deals are worth it, and how to prove it with `EXPLAIN` instead of guessing.*

Welcome to Week 6 of **C33 · Crunch SQL**. Weeks 1–3 made you fluent in querying; Week 4 taught you to model; Week 5 taught you correctness under concurrency. Now we make things *fast* — properly, with evidence. By the end of this week you can look at a slow query, pick the right index (and the right *type* of index), add it, and show the before/after in the query planner's own numbers. You will also know the harder half of the skill: when an index does nothing, when the planner refuses to use one, and when adding an index makes the whole system slower.

This is the hinge week of the course. Week 7 (reading plans and tuning) assumes everything here is second nature.

## The week's goal

**Pick the right index for a query, prove the win with `EXPLAIN (ANALYZE, BUFFERS)`, and recognize when an index hurts more than it helps.**

## Learning objectives

By the end of this week, you will be able to:

- **Explain** how a B-tree index is laid out on disk and why lookups, range scans, and `ORDER BY` all fall out of that one structure.
- **Predict** whether the planner will use an index, using two ideas: **selectivity** (how much of the table a predicate keeps) and **sargability** (whether a predicate can use an index at all).
- **Choose** among B-tree, hash, GIN, GiST, and BRIN for a given column and query shape — and justify the choice.
- **Design** composite indexes with the columns in the right order, plus **partial** and **covering** indexes, and trigger an **index-only scan**.
- **Measure** the cost of an index — write amplification, bloat, and maintenance — so you can argue for *removing* one.
- **Read** `EXPLAIN (ANALYZE, BUFFERS)` well enough to tell a `Seq Scan`, an `Index Scan`, an `Index Only Scan`, and a `Bitmap Heap Scan` apart and know why the planner picked each.

## Prerequisites

- Weeks 1–5 of C33 complete. You can write joins, aggregates, and CTEs, and you understand transactions and MVCC (indexes and MVCC interact — dead tuples cause index bloat).
- **PostgreSQL 16+** installed and running locally, with `psql` on your `PATH`. SQLite is used for a couple of contrast notes, but the deep material is Postgres.
- The **Week 6 seed dataset** loaded (a `shop` schema of customers, orders, and events). The setup script is in Lecture 1, §1, and every exercise reuses it.

## The seed dataset (used all week)

Every exercise and challenge runs against the same `shop` schema — a deliberately *large* fake e-commerce database so that index effects are visible in milliseconds instead of hiding under noise. Lecture 1 builds it:

- `customers` — ~100k rows, one per customer.
- `orders` — ~2 million rows, each linked to a customer, with a status, a total, and a `created_at`.
- `events` — ~2 million rows of semi-structured JSONB payloads (for the GIN lecture).

Two million rows is small for production but large enough that a sequential scan is clearly slower than an index lookup on your laptop.

## Weekly schedule

The schedule below adds up to approximately **28 hours** — the C33 full-time weekly target. Treat it as a guide, not a contract.

| Day       | Focus                                          | Lectures | Exercises | Challenges | Quiz/Read | Homework | Mini-Project | Daily Total |
|-----------|------------------------------------------------|---------:|----------:|-----------:|----------:|---------:|-------------:|------------:|
| Monday    | B-tree internals; selectivity & sargability    |    2h    |    1.5h   |     0h     |    0.5h   |   1h     |     0h       |     5h      |
| Tuesday   | Exercise 1 (B-tree before/after) + homework    |    0h    |    1.5h   |     1h     |    0.5h   |   1.5h   |     0h       |     4.5h    |
| Wednesday | The other index types (hash, GIN, GiST, BRIN)  |    2h    |    1.5h   |     0h     |    0.5h   |   1h     |     0h       |     5h      |
| Thursday  | Exercise 2 (GIN) + challenge 2 (unused index)  |    0h    |    1.5h   |     1.5h   |    0.5h   |   1h     |     0h       |     4.5h    |
| Friday    | Composite, partial, covering; the cost of index|    2h    |    1.5h   |     0h     |    0.5h   |   0h     |     0h       |     4h      |
| Saturday  | Mini-project (cut a slow query)                |    0h    |    0h     |     1h     |    0h     |   0h     |     3h       |     4h      |
| Sunday    | Quiz + reflection                              |    0h    |    0h     |     0h     |    1h     |   0h     |     0h       |     1h      |
| **Total** |                                                | **6h**   | **7.5h**  | **3.5h**   | **3.5h**  | **4.5h** | **3h**       | **28h**     |

## How to navigate this week

Number order is teaching order. Do the lectures before the exercises that lean on them.

| # | File | What's inside | Time |
|--:|------|---------------|-----:|
| 1 | [lecture-notes/01-how-a-btree-index-works.md](./lecture-notes/01-how-a-btree-index-works.md) | B-tree layout, when the planner uses it, selectivity & sargability | ~2h |
| 2 | [lecture-notes/02-the-other-index-types.md](./lecture-notes/02-the-other-index-types.md) | Hash, GIN (JSONB/FTS/arrays), GiST, BRIN — and when each wins | ~2h |
| 3 | [lecture-notes/03-composite-partial-covering-and-cost.md](./lecture-notes/03-composite-partial-covering-and-cost.md) | Column order, partial & covering indexes, index-only scans, the price of indexes | ~2h |
| 4 | [exercises/README.md](./exercises/README.md) | Index of the three exercises | — |
| 5 | [exercises/exercise-01-btree-before-after.md](./exercises/exercise-01-btree-before-after.md) | Add a B-tree, measure the win with `EXPLAIN (ANALYZE, BUFFERS)` | ~45m |
| 6 | [exercises/exercise-02-gin-jsonb-and-fts.md](./exercises/exercise-02-gin-jsonb-and-fts.md) | GIN for JSONB containment and full-text search | ~45m |
| 7 | [exercises/exercise-03-composite-partial-covering.md](./exercises/exercise-03-composite-partial-covering.md) | Composite column order, a partial index, and an index-only scan | ~45m |
| 8 | [challenges/README.md](./challenges/README.md) | Index of the two challenges | — |
| 9 | [challenges/challenge-01-cut-a-slow-query.md](./challenges/challenge-01-cut-a-slow-query.md) | Take a slow query to target with the right index | ~75m |
| 10 | [challenges/challenge-02-diagnose-unused-index.md](./challenges/challenge-02-diagnose-unused-index.md) | An index exists but the planner ignores it — find out why | ~60m |
| 11 | [mini-project/README.md](./mini-project/README.md) | Cut a slow query and document the before/after with plan evidence | ~3h |
| 12 | [homework.md](./homework.md) | Six practice problems reinforcing the week | ~4.5h |
| 13 | [quiz.md](./quiz.md) | 14 self-check questions with an answer key | ~1h |
| 14 | [resources.md](./resources.md) | Curated docs + the best external writing on indexing | — |

## By the end of this week you can…

- Read a query and predict which index (if any) the planner will pick — and be right most of the time.
- Add a B-tree, GIN, or BRIN index for the right reason and measure the improvement in real numbers.
- Put composite-index columns in the correct order and explain the "leftmost prefix" rule.
- Trigger an index-only scan and explain what made it possible.
- Argue, with evidence, for *dropping* an index that costs more than it returns.
- Spot the five classic reasons a "good" index goes unused.

## Up next

[Week 7 — Query optimization & plans](../week-07-query-optimization-and-plans/) — where you read whole plan trees and tune queries end to end. Indexing is half of that skill; this week is that half.

---

*Part of the Code Crunch Worldwide open curriculum · GPL-3.0 · If you find an error, open an issue or PR.*
