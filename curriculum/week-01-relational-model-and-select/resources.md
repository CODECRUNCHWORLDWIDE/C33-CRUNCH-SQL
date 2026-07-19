# Week 1 — Resources

Free, public, no signup unless noted. Read the "required" set; treat the rest as reference you dip into when a specific question comes up.

## Install first

- **PostgreSQL 16+** — the course's primary engine:
  <https://www.postgresql.org/download/> · macOS: [Postgres.app](https://postgresapp.com/) is the easiest. Linux: `sudo apt install postgresql` / `sudo dnf install postgresql-server`. Windows: the EDB installer.
- **SQLite 3.35+** — the zero-setup fallback; ships on macOS and most Linux already: <https://www.sqlite.org/download.html>. Check with `sqlite3 --version`.
- **A GUI (optional, later)** — [DBeaver](https://dbeaver.io/) (free, both engines) or [pgAdmin](https://www.pgadmin.org/) (Postgres). This week, stay in the terminal — it teaches you more.

## Required reading (this week's core)

- **PostgreSQL Tutorial, "The SQL Language"** — the gentle official intro; read chapters 2.1–2.6:
  <https://www.postgresql.org/docs/current/tutorial-sql.html>
  *Why: it's the canonical, correct explanation of `SELECT` from the people who built it.*
- **PostgreSQL — "Queries":** <https://www.postgresql.org/docs/current/queries.html>
  *Why: the reference for `WHERE`, `ORDER BY`, `LIMIT`, `DISTINCT` — bookmark it.*
- **SQLite — "Query Language Understood by SQLite":** <https://www.sqlite.org/lang.html>
  *Why: the other engine's contract; skim `SELECT` and the date functions.*
- **Modern SQL — "NULL":** <https://modern-sql.com/concept/null>
  *Why: the clearest short treatment of three-valued logic you'll find. Re-read before Challenge 2.*

## Reference (keep in tabs)

- **PostgreSQL — Functions and Operators** (string, math, date/time): <https://www.postgresql.org/docs/current/functions.html>
  *Why: when you need `ROUND`, `SUBSTRING`, or `EXTRACT` and forget the exact signature.*
- **PostgreSQL — Pattern Matching (`LIKE`/`ILIKE`/`SIMILAR TO`/regex):** <https://www.postgresql.org/docs/current/functions-matching.html>
  *Why: everything about wildcards and case-insensitive matching in one place.*
- **SQLite — Date and Time Functions:** <https://www.sqlite.org/lang_datefunc.html>
  *Why: the `strftime`/`date`/`julianday` reference you need when you're on SQLite instead of Postgres.*
- **SQLite — Datatypes and Type Affinity:** <https://www.sqlite.org/datatype3.html>
  *Why: explains SQLite's "lenient types" that surprise people coming from Postgres.*
- **`psql` cheat sheet (official docs):** <https://www.postgresql.org/docs/current/app-psql.html>
  *Why: all the `\d`, `\l`, `\x`, `\timing` meta-commands.*

## Practice beyond the seed table

- **PostgreSQL Exercises** — free, browser-based, graded `SELECT` drills: <https://pgexercises.com/>
  *Why: dozens more single-table and (later) join problems with instant checking.*
- **SQLBolt** — free interactive lessons, no install: <https://sqlbolt.com/>
  *Why: a good second pass on the same concepts with a different voice.*
- **SQLZoo** — classic free tutorial with quizzes: <https://sqlzoo.net/>
  *Why: more reps; the "SELECT basics" and "SELECT from WORLD" sections match this week.*

## Deeper background (optional this week)

- **Codd, "A Relational Model of Data for Large Shared Data Banks" (1970)** — the founding paper, surprisingly readable:
  <https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf>
  *Why: understand *why* tables-and-keys, not just *how*.*
- **"Use The Index, Luke"** — free book on how databases actually execute queries (you'll return to it in Weeks 6–7):
  <https://use-the-index-luke.com/>
  *Why: plants the seed for query performance; the preface alone is worth reading now.*

## Glossary

| Term | Definition |
|------|------------|
| **Relation / table** | A named set of rows sharing a fixed set of typed columns. |
| **Tuple / row** | One record — one employee. |
| **Attribute / column** | One property of every row, with a single data type. |
| **Domain / type** | The set of legal values for a column (integers, text, dates). |
| **Primary key** | Column(s) uniquely identifying a row; unique **and** not null. |
| **Foreign key** | Column referencing a key in another (or the same) table — how relationships are stored. |
| **RDBMS** | The software that stores tables and runs SQL (PostgreSQL, SQLite, …). |
| **Projection** | Choosing which columns to return (the `SELECT` list). |
| **Selection / filter** | Choosing which rows to return (the `WHERE` clause). |
| **Predicate** | A condition that evaluates to true / false / unknown. |
| **`NULL`** | The absence of a value — "unknown / not applicable." Not `0`, not `''`. |
| **Three-valued logic** | Boolean logic extended with "unknown"; `WHERE` keeps only `true` rows. |
| **Alias** | A rename of a column or table (`AS`) in the query output. |
| **`DISTINCT`** | Collapse duplicate result rows to one each. |
| **`LIMIT` / `OFFSET`** | Return at most N rows / skip the first M — pagination. |

---

*Broken link? Open an issue or PR.*
