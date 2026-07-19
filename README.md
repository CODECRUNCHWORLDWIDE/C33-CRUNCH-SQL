# C33 · Crunch SQL

> A free, open-source **12-week databases course** that takes you from your first `SELECT` to **senior/expert level**. From "what is a relational database?" to reading a query planner's output and tuning a slow production database to a latency target — relational modeling, joins and window functions, transactions and MVCC, index strategy, query optimization, stored procedures, security, and scaling with partitioning, replication, and sharding. The database foundation the backend, data, and cloud courses all assume.

[![License: GPL v3](https://img.shields.io/badge/License-GPL%20v3-blue.svg)](LICENSE)
[![PostgreSQL · SQLite](https://img.shields.io/badge/stack-PostgreSQL_·_SQLite-2463EB.svg)](#stack)
[![Built in the open](https://img.shields.io/badge/built-in%20the%20open-2463EB.svg)](https://github.com/CODECRUNCHWORLDWIDE)

C33 is a Tier-1 foundation course, sized long (12 weeks) because it goes all the way — beginner to expert. SQL is a *prerequisite* skill for [C16 Crunch Pro Web Backend](../C16-CRUNCH-PRO-WEB-BACKEND/), [C27 Crunch Data](../C27-CRUNCH-DATA/), and [C5 Crunch AI & Data Science](../C5-CRUNCH-AI-DATA-SCIENCE/) — this course makes you the person on the team who actually understands the database.

---

## Pathway summary

- **Full-time:** 12 weeks · ~28 hrs/week · ~336 hours
- **Working-engineer pace:** 6 months · ~14 hrs/week
- **Evening pace:** 12 months · ~7 hrs/week

See [`SYLLABUS.md`](SYLLABUS.md).

---

## What you will be able to do at the end of 12 weeks

- **Query fluently:** filter, sort, join (every kind), aggregate, and compose subqueries and CTEs without reaching for a tutorial.
- **Model data properly:** design a normalized schema (1NF → BCNF), choose keys and constraints, and know when to denormalize on purpose.
- **Reason about correctness under concurrency:** ACID, isolation levels, MVCC, locking, and how to avoid the anomalies each level allows.
- **Make queries fast:** pick the right index (B-tree, hash, GIN, GiST, BRIN, partial, covering, composite), read `EXPLAIN (ANALYZE, BUFFERS)`, and understand the planner's join algorithms, statistics, and cost model.
- **Write real database logic:** window functions and advanced analytics SQL, views, functions/stored procedures (PL/pgSQL), and triggers.
- **Secure and protect data:** roles and privileges, row-level security, and integrity constraints that hold at scale.
- **Scale past one box:** partitioning, replication, read replicas, sharding, and an honest SQL-vs-NoSQL decision.
- **Tune a real database:** take a slow, realistic workload and drive its latency down to a target — the capstone.

---

## Curriculum (12 weeks)

| Week | Topic | You leave able to… |
|------|-------|--------------------|
| 1 | [Relational model & SELECT](curriculum/week-01-relational-model-and-select/) | Query one table fluently — filter, sort, DISTINCT, LIMIT. |
| 2 | [Joins & set operations](curriculum/week-02-joins-and-set-operations/) | Combine tables with every join + UNION/INTERSECT/EXCEPT. |
| 3 | [Aggregation, subqueries & CTEs](curriculum/week-03-aggregation-subqueries-and-ctes/) | GROUP BY/HAVING, correlated subqueries, and readable CTEs. |
| 4 | [Data modeling & normalization](curriculum/week-04-data-modeling-and-normalization/) | Design a normalized schema with keys + constraints. |
| 5 | [Transactions, ACID & MVCC](curriculum/week-05-transactions-acid-and-mvcc/) | Choose isolation levels and avoid concurrency anomalies. |
| 6 | [Indexing deep-dive](curriculum/week-06-indexing-deep-dive/) | Pick the right index type and know when indexes hurt. |
| 7 | [Query optimization & plans](curriculum/week-07-query-optimization-and-plans/) | Read EXPLAIN/ANALYZE and tune a slow query. |
| 8 | [Window functions & advanced SQL](curriculum/week-08-window-functions-and-advanced-sql/) | Rank, run totals, and frame analytics in pure SQL. |
| 9 | [Procedures, functions & triggers](curriculum/week-09-procedures-functions-and-triggers/) | Write PL/pgSQL functions, views, and triggers. |
| 10 | [Security & data integrity](curriculum/week-10-security-and-data-integrity/) | Lock down access with roles + row-level security. |
| 11 | [Partitioning, replication & scaling](curriculum/week-11-partitioning-replication-and-scaling/) | Scale with partitioning, replicas, and sharding; SQL vs NoSQL. |
| 12 | [Capstone — tune & design](curriculum/week-12-capstone-tune-and-design/) | Design + tune a production database to a latency target. |

---

## How to navigate a week

Every week folder holds the same structure:

- **`README.md`** — the week overview + how the pieces fit + the week's goal.
- **`lecture-notes/`** — 3 lectures (~2 hrs each), the conceptual core.
- **`exercises/`** — 3 short, guided reps against a real database.
- **`challenges/`** — 2 open-ended problems with no single right answer.
- **`mini-project/`** — one build that ties the week together.
- **`homework.md`**, **`quiz.md`**, **`resources.md`** — practice, self-check, and further reading.

---

## Stack

PostgreSQL (16+) as the primary engine, with SQLite for zero-setup practice. Everything is free and runs on macOS, Linux, and Windows. You'll also use `psql`, `EXPLAIN`, and a seed dataset shipped with the course.

---

*Part of the Code Crunch Worldwide open curriculum · GPL-3.0 · [Browse all courses](https://codecrunchglobal.vercel.app/courses)*
