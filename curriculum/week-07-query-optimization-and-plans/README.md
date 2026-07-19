# Week 7 — Query Optimization & Plans

> **Goal:** Stop guessing why a query is slow. This week you learn to read `EXPLAIN (ANALYZE, BUFFERS)` like a senior engineer — the plan tree, the scan and join choices, the cost model behind them — and then use that reading to drive a real query from ten seconds down to under a hundred milliseconds, with evidence.

Week 6 gave you indexes. But an index the planner *refuses to use* is worthless, and an index it uses *wrongly* is a trap. The bridge between "I built an index" and "my query is fast" is the **query planner** — the part of PostgreSQL that turns your SQL into an execution strategy. This week you learn to see what it decided and why.

By Sunday you will be able to open any slow query, run one command, and answer three questions in order: *What is it actually doing? Why did the planner pick that? What one change makes it stop?* That skill — plan literacy — is what separates people who "know SQL" from people who own the database.

## Learning objectives

By the end of this week, you will be able to:

- **Read** a PostgreSQL plan tree: nodes, parent/child order, `cost`, `rows`, `width`, and the difference between *estimated* and *actual* rows and time.
- **Distinguish** `EXPLAIN` (estimate only) from `EXPLAIN (ANALYZE)` (really runs it) from `EXPLAIN (ANALYZE, BUFFERS)` (adds I/O), and know when each is safe to run.
- **Name** every scan type — sequential, index, index-only, bitmap heap/index — and say when the planner picks each.
- **Name** the three join algorithms — nested loop, hash join, merge join — and predict which one wins for a given query shape and data size.
- **Explain** the cost model: how `seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, and the row estimates combine into the number the planner minimizes.
- **Diagnose** a bad plan caused by stale or missing statistics, and fix it with `ANALYZE` or `CREATE STATISTICS`.
- **Rewrite** queries that defeat the planner — `NOT IN` with nullable columns, `OR` across columns, functions wrapped around indexed columns — into forms it can optimize.
- **Tune** `work_mem` and read when a sort or hash *spilled to disk* instead of staying in memory.
- **Prove** a speedup: capture a before-plan and an after-plan and show the latency drop.

## Prerequisites

- **Weeks 1–6 of C33** complete. You must be fluent with joins (Week 2), aggregation and CTEs (Week 3), and — critically — the indexing deep-dive (Week 6). This week assumes you know what a B-tree index *is*; it teaches you when the planner will *use* one.
- PostgreSQL 16+ installed locally, with `psql` working. SQLite 3.40+ for the contrast sections.
- The course seed dataset loaded (see `resources.md`). We use a deliberately large table this week — a few million rows — so the plans are realistic.

## Topics covered

- `EXPLAIN`, `EXPLAIN (ANALYZE)`, `EXPLAIN (ANALYZE, BUFFERS)`, and the `FORMAT` and `SETTINGS` options
- Reading the plan tree bottom-up: which node runs first, and what "rows" means at each level
- Estimated vs. actual rows — the single most important diagnostic in the whole plan
- Scan types: Seq Scan, Index Scan, Index Only Scan, Bitmap Heap + Bitmap Index Scan
- Join algorithms: Nested Loop, Hash Join, Merge Join — costs, memory, and when each wins
- The cost model and its cost constants; why `random_page_cost = 4` and what SSDs change
- Planner statistics: `ANALYZE`, `pg_stats`, `n_distinct`, `most_common_vals`, histograms
- When the planner goes wrong: stale stats, correlated columns, `OR`, `NOT IN`, non-sargable predicates
- Tuning knobs: `work_mem`, `default_statistics_target`, extended statistics (`CREATE STATISTICS`)
- How SQLite's planner differs (`EXPLAIN QUERY PLAN`, `ANALYZE`, the `sqlite_stat1` table)

## Weekly schedule

The schedule below adds up to approximately **28 hours** — this is an expert-tier week. Treat it as a target.

| Day       | Focus                                     | Lectures | Exercises | Challenges | Quiz/Read | Homework | Mini-Project | Daily Total |
|-----------|-------------------------------------------|---------:|----------:|-----------:|----------:|---------:|-------------:|------------:|
| Monday    | EXPLAIN + reading the plan tree           |    2h    |    1.5h   |     0h     |    0.5h   |   1h     |     0h       |     5h      |
| Tuesday   | Scans, joins, the cost model              |    2h    |    2h     |     0h     |    0.5h   |   1h     |     0h       |     5.5h    |
| Wednesday | Rewrites, stats, and the tuning knobs     |    2h    |    1.5h   |     1h     |    0.5h   |   1h     |     0h       |     6h      |
| Thursday  | Fix mis-estimated plans; join bake-off    |    0h    |    1h     |     1.5h   |    0.5h   |   1h     |     1h       |     5h      |
| Friday    | Mini-project — profile the slow query     |    0h    |    0h     |     0h     |    0h     |   0h     |     3h       |     3h      |
| Saturday  | Mini-project — tune to target + evidence  |    0h    |    0h     |     0h     |    0h     |   0h     |     2.5h     |     2.5h    |
| Sunday    | Quiz + reflection                         |    0h    |    0h     |     0h     |    1h     |   0h     |     0h       |     1h      |
| **Total** |                                           | **6h**   | **6h**    | **3.5h**   | **3h**    | **4h**   | **6.5h**     | **28h**     |

## How to navigate this week

Do the pieces in the numbered order below.

| # | File | What's inside | Time |
|--:|------|---------------|------|
| 1 | [lecture-notes/01-explain-and-reading-the-plan-tree.md](./lecture-notes/01-explain-and-reading-the-plan-tree.md) | `EXPLAIN` variants; the plan tree; estimated vs. actual | ~2h |
| 2 | [lecture-notes/02-scans-joins-and-the-cost-model.md](./lecture-notes/02-scans-joins-and-the-cost-model.md) | Scan types, join algorithms, the cost model, statistics | ~2h |
| 3 | [lecture-notes/03-rewriting-queries-and-tuning-knobs.md](./lecture-notes/03-rewriting-queries-and-tuning-knobs.md) | Rewrites, planner pitfalls, `work_mem`, extended stats | ~2h |
| 4 | [exercises/exercise-01-read-real-plans.md](./exercises/exercise-01-read-real-plans.md) | Interpret five real plans, in words | ~35m |
| 5 | [exercises/exercise-02-compare-join-algorithms.md](./exercises/exercise-02-compare-join-algorithms.md) | Force each join algorithm on one query; compare | ~40m |
| 6 | [exercises/exercise-03-fix-a-mis-estimated-plan.md](./exercises/exercise-03-fix-a-mis-estimated-plan.md) | Find the bad estimate; fix it with stats | ~40m |
| 7 | [challenges/challenge-01-ten-seconds-to-under-100ms.md](./challenges/challenge-01-ten-seconds-to-under-100ms.md) | Tune a 10s query under 100ms | ~1.5h |
| 8 | [challenges/challenge-02-why-the-plan-changed.md](./challenges/challenge-02-why-the-plan-changed.md) | Explain a plan flip after data grows | ~1h |
| 9 | [mini-project/README.md](./mini-project/README.md) | Profile + tune a slow query to target, with plan evidence | ~6.5h |
| 10 | [homework.md](./homework.md) | Six practice problems | ~4h |
| 11 | [quiz.md](./quiz.md) | 13 self-check questions + answer key | ~1h |
| 12 | [resources.md](./resources.md) | Official docs + the best plan-reading tools | — |

## By the end of this week you can…

- Run `EXPLAIN (ANALYZE, BUFFERS)` on any query and read the result out loud, top to bottom, without a reference open.
- Spot the one node where estimated rows and actual rows diverge by 100× — the usual root cause of a slow plan.
- Say *why* the planner chose a Seq Scan over your index, and whether it was right.
- Turn a ten-second query into a sub-hundred-millisecond query and produce the two plans that prove it.

---

*If you find errors, please open an issue or PR. Part of [C33 · Crunch SQL](../../README.md).*
