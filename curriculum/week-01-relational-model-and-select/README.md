# Week 1 — The Relational Model and `SELECT`

> **Goal:** by Sunday you can sit in front of one table and answer almost any question about it — filter it, sort it, deduplicate it, page through it — writing the SQL from memory, on both PostgreSQL and SQLite, without reaching for a tutorial.

Welcome to **C33 · Crunch SQL**. This is the week SQL stops being scary. A database is, at its heart, a set of tables; a table is a grid of rows and columns; and `SELECT` is the one verb you'll use ten thousand times to pull answers out of that grid. Get truly fluent with a single table now and every later week — joins, aggregation, windows, tuning — is just more of the same muscle.

We work against one seed table, `employees` (a 30-person fictional company called **Crunch**), for the whole week. You set it up once (below), then every lecture, exercise, challenge, and the mini-project queries it.

## Learning objectives

By the end of this week, you will be able to:

- **Explain** what a relational database *is* — tables, rows, columns, keys, the relational model, and what "RDBMS" means — and how PostgreSQL and SQLite differ.
- **Connect** to a database with `psql` (Postgres) and the `sqlite3` shell, and run your first `SELECT`.
- **Project** exactly the columns you want, compute expressions, and rename them with `AS` aliases.
- **Filter** rows with `WHERE` using comparisons, `AND`/`OR`/`NOT`, `IN`, `BETWEEN`, `LIKE`/`ILIKE`, and `IS NULL`.
- **Sort** with `ORDER BY` (multi-key, `ASC`/`DESC`, `NULLS FIRST/LAST`), remove duplicates with `DISTINCT`, and page results with `LIMIT`/`OFFSET`.
- **Reason** about SQL's three data types you meet first (text, numbers, dates), and about `NULL` and its three-valued logic — the single biggest source of "why is my query wrong?" bugs.
- **Reach** for common built-in functions: string (`LOWER`, `TRIM`, `||`, `SUBSTRING`), numeric (`ROUND`, `ABS`), and date (`CURRENT_DATE`, `EXTRACT`, `AGE`).

## Prerequisites

- You can run commands in a terminal (open one, type a command, read the output).
- **No** prior SQL or database knowledge. That's what this course is for.
- PostgreSQL 16+ **or** SQLite 3.35+ installed. Both are free. Install steps are in [`resources.md`](./resources.md). If you install only one, install **PostgreSQL** — it's the primary engine for the course; SQLite is the zero-setup fallback.

## Set up the seed database (do this first)

Everything this week runs against one table. Create it once.

**PostgreSQL:**

```bash
createdb crunch          # make a database named "crunch"
psql crunch              # open a session against it
```

**SQLite:**

```bash
sqlite3 crunch.db        # creates the file on first write
```

Then paste this into the shell (it works unchanged on both engines):

```sql
CREATE TABLE employees (
    emp_id         INTEGER PRIMARY KEY,
    first_name     TEXT    NOT NULL,
    last_name      TEXT    NOT NULL,
    email          TEXT,               -- nullable on purpose
    department     TEXT    NOT NULL,
    job_title      TEXT    NOT NULL,
    salary         NUMERIC NOT NULL,
    hire_date      DATE    NOT NULL,
    birth_date     DATE,
    city           TEXT,
    country        TEXT,
    is_remote      BOOLEAN NOT NULL,
    commission_pct NUMERIC,            -- only Sales earns commission; NULL elsewhere
    manager_id     INTEGER             -- NULL for the CEO
);

INSERT INTO employees VALUES
(1,'Grace','Hopper','grace.hopper@crunch.io','Executive','CEO',320000,'2014-01-06','1972-05-20','Austin','USA',FALSE,NULL,NULL),
(2,'Ada','Lovelace','ada.lovelace@crunch.io','Engineering','VP Engineering',240000,'2015-03-01','1979-12-10','Austin','USA',FALSE,NULL,1),
(3,'Alan','Turing','alan.turing@crunch.io','Engineering','Staff Engineer',190000,'2016-07-15','1983-06-23','Seattle','USA',TRUE,NULL,2),
(4,'Linus','Torvalds','linus.torvalds@crunch.io','Engineering','Senior Engineer',165000,'2017-02-20','1985-12-28','Toronto','Canada',TRUE,NULL,2),
(5,'Margaret','Hamilton','margaret.hamilton@crunch.io','Engineering','Senior Engineer',168000,'2016-11-03','1986-08-17','Seattle','USA',FALSE,NULL,2),
(6,'Katherine','Johnson','katherine.johnson@crunch.io','Engineering','Engineer',132000,'2019-05-13','1990-08-26','Austin','USA',TRUE,NULL,3),
(7,'Dennis','Ritchie','dennis.ritchie@crunch.io','Engineering','Engineer',128000,'2020-09-01','1991-09-09','London','UK',TRUE,NULL,3),
(8,'Barbara','Liskov','barbara.liskov@crunch.io','Engineering','Junior Engineer',98000,'2022-06-06','1996-11-07','Berlin','Germany',TRUE,NULL,4),
(9,'Don','Draper','don.draper@crunch.io','Sales','VP Sales',210000,'2015-08-10','1978-03-15','Miami','USA',FALSE,0.15,1),
(10,'Peggy','Olson','peggy.olson@crunch.io','Sales','Account Executive',120000,'2018-04-02','1988-05-25','Miami','USA',FALSE,0.12,9),
(11,'Pete','Campbell','pete.campbell@crunch.io','Sales','Account Executive',118000,'2018-09-17','1989-01-30','Miami','USA',TRUE,0.12,9),
(12,'Joan','Holloway','joan.holloway@crunch.io','Sales','Sales Manager',145000,'2016-05-19','1984-07-11','Miami','USA',FALSE,0.10,9),
(13,'Ken','Cosgrove','ken.cosgrove@crunch.io','Sales','Sales Rep',92000,'2021-01-11','1993-02-14','London','UK',TRUE,0.08,12),
(14,'Harry','Crane','harry.crane@crunch.io','Sales','Sales Rep',88000,'2021-11-29','1994-10-05','Madrid','Spain',TRUE,0.08,12),
(15,'Sofia','Garcia','sofia.garcia@crunch.io','Marketing','Marketing Director',155000,'2017-03-27','1987-04-19','Madrid','Spain',FALSE,NULL,1),
(16,'Liam','Murphy','liam.murphy@crunch.io','Marketing','Content Lead',105000,'2019-08-08','1991-12-01','Toronto','Canada',TRUE,NULL,15),
(17,'Noah','Silva','noah.silva@crunch.io','Marketing','Marketing Analyst',82000,'2022-02-14','1997-03-22','Sao Paulo','Brazil',TRUE,NULL,15),
(18,'Emma','Schmidt',NULL,'Marketing','Designer',79000,'2023-01-09','1998-06-30','Berlin','Germany',TRUE,NULL,15),
(19,'Olivia','Rossi','olivia.rossi@crunch.io','Support','Support Manager',112000,'2018-06-25','1988-09-14','Miami','USA',FALSE,NULL,1),
(20,'James','Anderson','james.anderson@crunch.io','Support','Support Engineer',76000,'2020-10-19','1992-11-11','Austin','USA',TRUE,NULL,19),
(21,'Priya','Patel','priya.patel@crunch.io','Support','Support Engineer',74000,'2021-07-05','1995-05-08','Bengaluru','India',TRUE,NULL,19),
(22,'Arjun','Sharma','arjun.sharma@crunch.io','Support','Support Rep',58000,'2023-03-20','1999-02-28','Bengaluru','India',TRUE,NULL,19),
(23,'William','Brown','william.brown@crunch.io','Finance','CFO',260000,'2014-09-15','1975-01-25','Austin','USA',FALSE,NULL,1),
(24,'Isabella','Costa','isabella.costa@crunch.io','Finance','Accountant',95000,'2019-12-02','1990-07-19','Sao Paulo','Brazil',FALSE,NULL,23),
(25,'Lucas','Meyer','lucas.meyer@crunch.io','Finance','Financial Analyst',89000,'2021-04-12','1993-10-03','Berlin','Germany',TRUE,NULL,23),
(26,'Mia','Johansson','mia.johansson@crunch.io','HR','HR Director',140000,'2016-02-29','1985-03-08','London','UK',FALSE,NULL,1),
(27,'Ethan','Wilson','ethan.wilson@crunch.io','HR','Recruiter',72000,'2022-08-15','1996-04-17','Toronto','Canada',TRUE,NULL,26),
(28,'Charlotte','Dubois','charlotte.dubois@crunch.io','Operations','Ops Manager',118000,'2017-10-30','1986-12-12','London','UK',FALSE,NULL,1),
(29,'Benjamin','Klein',NULL,'Operations','Ops Analyst',84000,'2020-03-16','1992-08-21','Berlin','Germany',TRUE,NULL,28),
(30,'Amelia','Novak','amelia.novak@crunch.io','Operations','Logistics Coordinator',68000,'2023-06-01','2000-01-15','Madrid','Spain',TRUE,NULL,28);
```

Sanity check — this should print `30`:

```sql
SELECT COUNT(*) FROM employees;
```

Two rows have a `NULL` email (18, 29) and every non-Sales row has a `NULL` `commission_pct`. Those `NULL`s are there **on purpose** — you'll need them for the pattern-matching and three-valued-logic lessons.

## Weekly schedule

The schedule below adds up to approximately **28 hours** (the course's full-time pace). Treat it as a target.

| Day | Focus | Lectures | Exercises | Challenges | Quiz/Read | Homework | Mini-Project | Daily Total |
|-----------|------------------------------------------|---------:|----------:|-----------:|----------:|---------:|-------------:|------------:|
| Monday | Install + seed; what a database *is* | 2h | 1h | 0h | 0.5h | 1h | 0h | 4.5h |
| Tuesday | `SELECT`, columns, aliases, `WHERE` | 2h | 1.5h | 0h | 0.5h | 1h | 0h | 5h |
| Wednesday | Pattern matching, `IN`/`BETWEEN`, `NULL` | 2h | 1.5h | 1h | 0.5h | 1h | 0h | 6h |
| Thursday | Sorting, `DISTINCT`, `LIMIT`/`OFFSET` | 0h | 1.5h | 1h | 0.5h | 1h | 1h | 5h |
| Friday | Types + functions; challenges | 0h | 0h | 1h | 0.5h | 1h | 1.5h | 4h |
| Saturday | Mini-project (20 business questions) | 0h | 0h | 0h | 0h | 0h | 2.5h | 2.5h |
| Sunday | Quiz + review | 0h | 0h | 0h | 1h | 0h | 0h | 1h |
| **Total** | | **6h** | **5.5h** | **3h** | **3.5h** | **5h** | **5h** | **28h** |

## How to navigate this week

Work top to bottom. Each piece assumes the ones above it.

| # | File | What's inside | ~Time |
|--:|------|---------------|------:|
| 1 | [lecture-notes/01-what-is-a-relational-database.md](./lecture-notes/01-what-is-a-relational-database.md) | Tables/rows/columns, keys, the relational model, RDBMS, Postgres vs SQLite, connecting with `psql` | 2h |
| 2 | [lecture-notes/02-select-deep.md](./lecture-notes/02-select-deep.md) | Columns, expressions, aliases, `WHERE`, `ORDER BY`, `DISTINCT`, `LIMIT`/`OFFSET` | 2h |
| 3 | [lecture-notes/03-data-types-null-and-functions.md](./lecture-notes/03-data-types-null-and-functions.md) | Data types, `NULL` + three-valued logic, string/number/date functions | 2h |
| 4 | [exercises/exercise-01-first-queries.md](./exercises/exercise-01-first-queries.md) | Your first `SELECT`s against the seed table | 1h |
| 5 | [exercises/exercise-02-filtering-and-pattern-matching.md](./exercises/exercise-02-filtering-and-pattern-matching.md) | `WHERE`, `LIKE`/`ILIKE`, `IN`, `BETWEEN`, `IS NULL` | 1h |
| 6 | [exercises/exercise-03-sorting-dedup-pagination.md](./exercises/exercise-03-sorting-dedup-pagination.md) | `ORDER BY`, `DISTINCT`, `LIMIT`/`OFFSET` | 1h |
| 7 | [challenges/challenge-01-ambiguous-business-questions.md](./challenges/challenge-01-ambiguous-business-questions.md) | Turn fuzzy English into precise `WHERE` clauses | 1h |
| 8 | [challenges/challenge-02-null-gotchas.md](./challenges/challenge-02-null-gotchas.md) | Find and fix the `NULL` traps | 1h |
| 9 | [mini-project/README.md](./mini-project/README.md) | Answer 20 business questions from one table | 2.5h |
| 10 | [homework.md](./homework.md) | Extra practice, spaced out | 5h |
| 11 | [quiz.md](./quiz.md) | 14 self-check questions + answer key | 1h |
| 12 | [resources.md](./resources.md) | Official docs + the few links worth your time | — |

## By the end of this week you can…

- Read a table's shape and describe its columns, types, and keys.
- Write a `SELECT` that projects, filters, sorts, dedups, and pages — from memory.
- Predict what `NULL` will do *before* it surprises you.
- Answer a real business question posed in plain English by translating it into precise SQL.

## Up next

[Week 2 — Joins & set operations](../week-02-joins-and-set-operations/) — once one table feels easy, we combine many.

---

*Part of the Code Crunch Worldwide open curriculum · GPL-3.0 · If you find errors, please open an issue or PR.*
