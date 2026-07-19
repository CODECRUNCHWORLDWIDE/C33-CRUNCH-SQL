# Week 12 — Capstone: Tune & Design

> **Goal:** Take a slow, seeded production-shaped database and drive it to a stated latency target — by measuring first, reading the query plan, indexing and rewriting deliberately, and verifying every change with before/after `EXPLAIN` plans. Then design a fresh schema that would not have been slow in the first place.

This is the last week of C33. There is nothing new to memorize — everything you need you already met in Weeks 1–11. What is new is the **discipline**: a senior engineer does not "add an index and hope." They form a hypothesis, measure it, change one thing, and prove the change worked. This week you build that reflex on a database that is deliberately, realistically slow.

You will finish with a written tuning report — the same artifact you would hand a tech lead in a real job — containing before/after plans, the exact changes you made, and *why* each one helped.

## Learning objectives

By the end of this week, you will be able to:

- **Profile** a running database to find its worst queries using `pg_stat_statements`, `EXPLAIN (ANALYZE, BUFFERS)`, and `auto_explain` — measuring before touching anything.
- **Read** a plan tree well enough to name the bottleneck: a `Seq Scan` that should be an index scan, a bad row estimate, a spilling sort, or an N+1 pattern hiding in the application.
- **Fix** deliberately — the right index (composite, partial, covering), a query rewrite, a data-type correction, or a statistics refresh — and know which lever to pull first.
- **Verify** with a repeatable before/after comparison, and quote a *p95 latency* number, not just a single lucky run.
- **Review** a schema for the common anti-patterns — EAV, over-indexing, wrong types, N+1, unbounded `SELECT *` — and articulate the trade-off each one makes.
- **Plan capacity**: estimate table and index growth, decide what to monitor, and know when a query is fast enough to *stop* tuning.
- **Design** a production schema from a product description that meets a latency budget by construction.

## Prerequisites

- **C33 Weeks 1–11** complete. This week leans hardest on Week 6 (indexing), Week 7 (plans), and Week 4 (modeling). Keep those notes open.
- **PostgreSQL 16+** installed locally, with the `pg_stat_statements` extension available (ships with `postgresql-contrib`).
- **SQLite 3.40+** for the zero-setup comparisons.
- Comfortable running `psql`, writing DDL, and reading `EXPLAIN` output. If any of those is shaky, review before starting — the capstone assumes fluency.

## Topics covered

- The measure → read → change → verify → repeat loop, and why skipping "measure" wastes days.
- `pg_stat_statements`: total time vs. mean time vs. call count — which query is *actually* the problem.
- The bottleneck vocabulary: seq scan, nested loop vs. hash join, sort spill, bad estimate, buffer reads.
- Schema anti-patterns: EAV, polymorphic keys, `text` for everything, over-indexing on write-heavy tables, and the N+1 that lives in the ORM.
- Capacity math: rows × row-width, index bloat, autovacuum, and back-of-envelope growth projection.
- Knowing when to stop: latency budgets, the p95/p99 distinction, and the cost of one more index.

## Weekly schedule

The schedule below adds up to approximately **30 hours**. This is an expert week — the mini-project is a real deliverable, so more time sits in the project than in a normal week.

| Day       | Focus                                        | Lectures | Exercises | Challenges | Quiz/Read | Homework | Mini-Project | Daily Total |
|-----------|----------------------------------------------|---------:|----------:|-----------:|----------:|---------:|-------------:|------------:|
| Monday    | Tuning methodology; set up the slow DB       |    2h    |    1h     |     0h     |    0.5h   |   1h     |     0h       |    4.5h     |
| Tuesday   | Profiling — find the worst queries           |    0h    |    2h     |     0h     |    0.5h   |   1h     |     0h       |    3.5h     |
| Wednesday | Schema-design review + anti-patterns         |    2h    |    1.5h   |     1h     |    0.5h   |   1h     |     0h       |    6h       |
| Thursday  | Apply fixes; verify with before/after plans  |    0h    |    2h     |     1h     |    0.5h   |   1h     |     1h       |    5.5h     |
| Friday    | Capacity planning, monitoring, when to stop  |    2h    |    0h     |     0h     |    0.5h   |   1h     |     2h       |    5.5h     |
| Saturday  | Capstone build                               |    0h    |    0h     |     0h     |    0h     |   0h     |     4h       |    4h       |
| Sunday    | Quiz + write the report                      |    0h    |    0h     |     0h     |    0.5h   |   0h     |     0.5h     |    1h       |
| **Total** |                                              | **6h**   | **8.5h**  | **2h**     | **3h**    | **5h**   | **7.5h**     | **30h**     |

## How to navigate this week

Work in teaching order, top to bottom.

| # | File | What's inside | Time |
|---|------|---------------|-----:|
| 1 | [lecture-notes/01-a-performance-tuning-methodology.md](./lecture-notes/01-a-performance-tuning-methodology.md) | The measure → read → change → verify loop, in full | 2h |
| 2 | [lecture-notes/02-schema-design-review-anti-patterns.md](./lecture-notes/02-schema-design-review-anti-patterns.md) | EAV, over-indexing, wrong types, N+1 — and the trade-offs | 2h |
| 3 | [lecture-notes/03-capacity-planning-and-monitoring.md](./lecture-notes/03-capacity-planning-and-monitoring.md) | Growth math, `pg_stat_statements`, knowing when to stop | 2h |
| 4 | [exercises/README.md](./exercises/README.md) | Index of the three guided reps | — |
| 5 | [exercises/exercise-01-profile-a-slow-database.md](./exercises/exercise-01-profile-a-slow-database.md) | Seed a slow DB, find its worst queries | 2h |
| 6 | [exercises/exercise-02-propose-and-apply-fixes.md](./exercises/exercise-02-propose-and-apply-fixes.md) | Propose fixes, apply them one at a time | 2h |
| 7 | [exercises/exercise-03-verify-with-before-after-plans.md](./exercises/exercise-03-verify-with-before-after-plans.md) | Prove each fix with a before/after plan | 1.5h |
| 8 | [challenges/README.md](./challenges/README.md) | Index of the two stretch challenges | — |
| 9 | [challenges/challenge-01-design-a-production-schema.md](./challenges/challenge-01-design-a-production-schema.md) | Design a schema for a described product | 1h |
| 10 | [challenges/challenge-02-defend-a-tuning-decision.md](./challenges/challenge-02-defend-a-tuning-decision.md) | Defend a decision against a skeptical reviewer | 1h |
| 11 | [mini-project/README.md](./mini-project/README.md) | **CAPSTONE** — tune a DB to a latency target, write the report | 7.5h |
| 12 | [homework.md](./homework.md) | Six practice tasks reinforcing the week | 5h |
| 13 | [quiz.md](./quiz.md) | 12 self-check questions + answer key | 0.5h |
| 14 | [resources.md](./resources.md) | Official docs + curated further reading | — |

## By the end of this week you can…

- Walk into a slow database, and within an hour name the three queries costing the most and *why*.
- Apply a fix and prove — with a plan and a p95 number — that it worked.
- Spot EAV, over-indexing, and an ORM N+1 in a schema review without running anything.
- Design a schema that meets a latency budget on the first try.
- Write the tuning report that gets you taken seriously as the person who understands the database.

## Up next

Nothing — this is the finish line for C33. From here, the database chops you built feed directly into [C16 Crunch Pro Web Backend](../../../C16-CRUNCH-PRO-WEB-BACKEND/), [C27 Crunch Data](../../../C27-CRUNCH-DATA/), and [C5 Crunch AI & Data Science](../../../C5-CRUNCH-AI-DATA-SCIENCE/). Ship the capstone report and add it to your portfolio — it is the single most convincing artifact in this course.

---

*If you find errors, please open an issue or PR. Part of Code Crunch Worldwide · GPL-3.0.*
