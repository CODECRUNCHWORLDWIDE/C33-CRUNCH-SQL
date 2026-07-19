# Week 4 — Data Modeling & Normalization

> **Goal:** Design a normalized relational schema from a written spec — choose the right keys, model 1:1 / 1:N / M:N relationships, walk a design up to 3NF (and recognize BCNF), and build it as fully-constrained, runnable DDL. By Sunday you can turn a paragraph of English into a database that *enforces its own rules*.

Weeks 1–3 taught you to *query* schemas other people built. This week you become the designer. A good model makes every future query short and correct; a bad one makes you fight your own database forever. The skill has three moves — **design** the entities and relationships, **normalize** away redundancy, and **build** with constraints — and this week drills all three, ending with a real schema you design and create from scratch.

## Learning objectives

By the end of this week, you will be able to:

- **Choose** the right key for every table — primary, candidate, and foreign keys, and surrogate vs. natural — and say *why*.
- **Model** all three cardinalities (1:1, 1:N, M:N), and resolve every many-to-many with a junction table.
- **Draw** an entity-relationship diagram a teammate can turn into `CREATE TABLE`s.
- **Normalize** a messy wide table step by step — 1NF → 2NF → 3NF → BCNF — naming the functional dependency and the anomaly removed at each step.
- **Write** DDL with the full constraint toolkit: NOT NULL, UNIQUE, CHECK, PRIMARY KEY, FOREIGN KEY with the correct `ON DELETE` action.
- **Evolve** a live schema with `ALTER TABLE`, and decide — with reasons — when to *denormalize* on purpose.

## Prerequisites

- **C33 Weeks 1–3** completed — you can `SELECT`, join, aggregate, and read a subquery/CTE.
- **PostgreSQL 16** installed (primary engine this week — it actually enforces types and constraints), and/or **SQLite** for zero-setup practice. Setup commands are in [`exercises/README.md`](./exercises/README.md).

## Stack note

We default to **PostgreSQL 16** for all schema work because it enforces types and constraints strictly. SQLite examples are included, but SQLite needs `PRAGMA foreign_keys = ON;` per connection and `STRICT` tables to enforce what Postgres enforces by default — a difference the lectures call out where it matters.

## How to navigate this week

Work top to bottom — the pieces are numbered in teaching order (design → normalize → build).

| # | File | What's inside | Time |
|--:|------|---------------|-----:|
| 1 | [lecture-notes/01-keys-relationships-and-er-modeling.md](./lecture-notes/01-keys-relationships-and-er-modeling.md) | Every kind of key; 1:1/1:N/M:N; junction tables; ER diagrams | 2h |
| 2 | [lecture-notes/02-normalization-1nf-to-bcnf.md](./lecture-notes/02-normalization-1nf-to-bcnf.md) | The three anomalies, functional dependencies, 1NF→BCNF with worked examples | 2.5h |
| 3 | [lecture-notes/03-ddl-constraints-and-denormalization.md](./lecture-notes/03-ddl-constraints-and-denormalization.md) | `CREATE`/`ALTER TABLE`, all five constraints, `ON DELETE`, denormalization trade-offs | 2h |
| 4 | [exercises/exercise-01-draw-an-er-model.md](./exercises/exercise-01-draw-an-er-model.md) | Model a library domain as an ER diagram with keys + cardinality | 0.7h |
| 5 | [exercises/exercise-02-normalize-a-wide-table.md](./exercises/exercise-02-normalize-a-wide-table.md) | Walk a messy conference table 1NF → 3NF, step by step | 0.75h |
| 6 | [exercises/exercise-03-write-the-ddl.md](./exercises/exercise-03-write-the-ddl.md) | Build the library schema with full constraints; prove each one bites | 0.85h |
| 7 | [challenges/challenge-01-design-from-a-spec.md](./challenges/challenge-01-design-from-a-spec.md) | Design a clinic-appointment schema from a written spec | 2.5h |
| 8 | [challenges/challenge-02-spot-and-fix-violations.md](./challenges/challenge-02-spot-and-fix-violations.md) | Audit a "god-table" and repair every violation | 2h |
| 9 | [mini-project/README.md](./mini-project/README.md) | Design + create a normalized "Crunch Eats" schema from a spec | 5–6h |
| — | [homework.md](./homework.md) | Six practice problems (keys, cardinality, normalization, constraints) | ~5h |
| — | [quiz.md](./quiz.md) | 13 self-check questions with answer key | 0.5h |
| — | [resources.md](./resources.md) | Curated PostgreSQL docs + normalization references | — |

## Suggested weekly schedule (~28h)

| Day | Focus | Hours |
|-----|-------|------:|
| Mon | Lecture 1 (keys, relationships, ER) + Exercise 1 | 3.5 |
| Tue | Lecture 2 (normalization) + Exercise 2 | 3.5 |
| Wed | Lecture 3 (DDL & constraints) + Exercise 3 | 3.5 |
| Thu | Homework problems 1–4 | 4 |
| Fri | Challenges 1 & 2 + homework 5–6 | 6 |
| Sat | Mini-project (Crunch Eats schema) | 6 |
| Sun | Quiz + review + commit | 1.5 |

## By the end of this week you can…

- Read a paragraph of business requirements and list the entities, attributes, and relationships hiding in it.
- Pick a primary key on purpose (and know when a surrogate beats a natural key — and what you must *still* enforce).
- Resolve any many-to-many with a junction table, and put relationship attributes in the right place.
- Take a wide, redundant table and normalize it to 3NF, naming each anomaly you kill.
- Write `CREATE TABLE` DDL where illegal data is *impossible*, not merely discouraged — and choose the right `ON DELETE` for every foreign key.
- Explain when and how to denormalize without lying to yourself about the cost.

## Up next

[Week 5 — Transactions, ACID & MVCC](../week-05-transactions-acid-and-mvcc/) — now that your schema is correct *at rest*, make it correct *under concurrency*.

---

*Part of [C33 · Crunch SQL](../../README.md) · GPL-3.0 · If you find an error, open an issue or PR.*
