# Lecture 1 — What a Relational Database Is

> **Duration:** ~2 hours. **Outcome:** You can explain what a relational database is — tables, rows, columns, keys, the relational model behind it, and what "RDBMS" means — tell PostgreSQL and SQLite apart, and connect to a live database with `psql` to run your first `SELECT`.

Before you write a single query, you should be able to answer "what am I even querying?" Almost everyone learns SQL syntax first and the *model* never. That's backwards, and it's why so many people write SQL that technically runs but quietly returns the wrong rows. This lecture builds the model first. The syntax in Lecture 2 will then feel obvious.

## 1. Why databases exist

You could store your company's data in a spreadsheet, or a pile of CSV files, or a folder of JSON. People do. It works until it doesn't, and it stops working the moment you have:

- **More than one person** reading and writing at the same time.
- **More data than fits in memory**, so you need to fetch *just* the rows you want.
- **Rules that must always hold** — "every employee has a department," "no two employees share an ID" — even when ten programs are writing at once.
- **Questions you didn't anticipate** — "average salary of remote engineers hired after 2020" — that you want answered in milliseconds, not by rewriting code.

A **database** is software built to store data durably, let many clients read and write it safely and concurrently, enforce rules about it, and answer arbitrary questions about it *fast*. A **relational** database is the dominant kind, and has been for 50 years, because the model underneath it is simple and provably powerful.

## 2. The relational model

In 1970, Edgar F. Codd proposed organizing data as **relations** — what we informally call **tables**. The whole model is just a handful of ideas:

| Term (formal) | Term (everyday) | What it is |
|---------------|-----------------|------------|
| Relation | Table | A named set of rows with a fixed set of columns |
| Tuple | Row / record | One entity — one employee, one order |
| Attribute | Column / field | One property — `salary`, `hire_date` |
| Domain | Data type | The set of legal values for a column (integers, text, dates) |
| Cardinality | Row count | How many rows the table has |
| Degree | Column count | How many columns the table has |

A table is a **grid**: columns across the top define *what* you know about each thing; rows going down are the individual things. Here are the first few rows of our seed table, `employees`:

```
 emp_id | first_name | last_name | department  |  salary  | is_remote
--------+------------+-----------+-------------+----------+-----------
      1 | Grace      | Hopper    | Executive   | 320000   | f
      2 | Ada        | Lovelace  | Engineering | 240000   | f
      3 | Alan       | Turing    | Engineering | 190000   | t
```

Two properties of the model matter enormously and trip up beginners:

1. **A table is an (unordered) set of rows.** There is *no* inherent first row or last row. If you want an order, you must ask for one with `ORDER BY` (Lecture 2). The database is free to hand rows back in any order otherwise, and it will change between runs.
2. **Every value in a column shares one type (domain).** `salary` is numbers, all the way down. You can't put "sometime next year" in a `DATE` column. This is what makes queries reliable.

Codd's insight was that if you model data this way, you can answer *any* question with a small, closed algebra of operations (select rows, project columns, join tables, union, etc.). SQL is a practical language built on top of that algebra. Everything you learn this course — every join, aggregate, and window function — is a combination of those primitive operations.

## 3. Keys — how rows are identified and connected

If a table is a set of rows, how do you point at *one specific* row? With a **key**.

- A **primary key (PK)** is a column (or set of columns) whose value is **unique** and **never NULL** — it identifies each row unambiguously. In `employees`, `emp_id` is the primary key. There is exactly one row where `emp_id = 12`, forever.
- A **natural key** is a PK made of real-world data (e.g., an email address, a country code). A **surrogate key** is an artificial one with no meaning outside the database (an auto-incrementing integer like `emp_id`). Surrogates are usually safer — real-world "unique" facts have a habit of turning out not to be.
- A **foreign key (FK)** is a column that *references* the primary key of another (or the same) table. In `employees`, `manager_id` is a foreign key pointing back at `emp_id` — employee 3's `manager_id` is `2`, meaning "Alan's manager is Ada." This is how relational databases represent relationships: **by matching a value in one row to a key in another.** That's the "relational" in relational database. Week 2 (joins) is entirely about following these links.

A table can also have a **unique constraint** on a non-PK column (e.g., "email must be unique") and rows can have **NULL** in a nullable column (email is nullable in our table — two employees have none). Keep the distinction crisp: *primary key* = unique **and** not null; *unique constraint* = unique but may allow one NULL; *foreign key* = must match a key elsewhere.

We design keys and constraints properly in Week 4. For now, just know that `emp_id` is the handle for a row and `manager_id` links a row to another.

## 4. RDBMS — the software around the model

A **Relational Database Management System (RDBMS)** is the actual program that stores your tables on disk, keeps them consistent, and runs your queries. The model is the theory; the RDBMS is the engine. Popular ones: **PostgreSQL**, **MySQL/MariaDB**, **SQLite**, **SQL Server**, **Oracle**. They all speak **SQL** (Structured Query Language), the standard language for talking to relational databases — but each dialect has quirks.

An RDBMS gives you, roughly:

- **A storage engine** — how bytes hit the disk, in pages, with a write-ahead log so a crash doesn't corrupt your data.
- **A query planner/optimizer** — you say *what* rows you want (declarative SQL); it decides *how* to fetch them fast (Week 7).
- **A transaction system** — groups of changes that succeed or fail as a unit, with ACID guarantees (Week 5).
- **A client/server protocol** (usually) — programs connect over a socket and send SQL.

SQL is **declarative**: you describe the result you want, not the steps to compute it. `SELECT last_name FROM employees WHERE salary > 100000` says *what* — "the surnames of people paid over 100k" — and the engine figures out the *how*. This is the mental shift from ordinary programming, and it's freeing once it clicks.

## 5. PostgreSQL vs SQLite — the two engines this course uses

We use two engines on purpose. They anchor the two ends of the spectrum.

| | **PostgreSQL** | **SQLite** |
|--|----------------|------------|
| Shape | Client/server — a running daemon you connect to | A library — the "database" is a single file |
| Setup | Install + a running service | Zero: `sqlite3 file.db` and you're in |
| Concurrency | Many simultaneous readers **and** writers | Many readers, one writer at a time |
| Type system | Strict, rich (arrays, JSON, ranges, custom types) | Flexible "type affinity" — lenient by design |
| Use it for | Production apps, this course's primary engine | Local dev, phones, tests, embedded, quick practice |
| Case-insensitive match | `ILIKE` | `LIKE` is already case-insensitive for ASCII |

The important message: **99% of what you learn is identical on both.** `SELECT ... WHERE ... ORDER BY ...` is byte-for-byte the same. The differences are at the edges (Postgres has `ILIKE`; SQLite's `LIKE` is already case-insensitive; Postgres enforces types strictly, SQLite is lenient). We'll call out the differences the moment they matter. Default to PostgreSQL; fall back to SQLite when you want zero setup.

SQLite, by the way, is not a toy — it's the most widely deployed database in the world (every phone, browser, and countless apps ship it). "Small footprint" is not "small importance."

## 6. Connecting and running your first query

### PostgreSQL with `psql`

`psql` is Postgres's interactive terminal. Create a database and connect:

```bash
createdb crunch      # one-time: make a database named "crunch"
psql crunch          # open an interactive session
```

You'll get a prompt like `crunch=#`. Useful **meta-commands** (they start with a backslash and are `psql` features, *not* SQL):

```
\l           list all databases
\dt          list tables in the current database
\d employees describe the employees table (columns + types)
\x           toggle "expanded" display (great for wide rows)
\timing      show how long each query takes
\q           quit
```

Run your first real query (SQL statements end with a semicolon):

```sql
SELECT first_name, last_name, department
FROM employees
WHERE department = 'Engineering';
```

### SQLite with `sqlite3`

```bash
sqlite3 crunch.db    # opens (and creates) the file
```

Prompt is `sqlite>`. Its **dot-commands** mirror psql's meta-commands:

```
.tables            list tables
.schema employees  show the CREATE TABLE statement
.headers on        print column names above results
.mode column       align output in columns (much more readable)
.quit              exit
```

Run the *same* SQL — it's identical:

```sql
SELECT first_name, last_name, department
FROM employees
WHERE department = 'Engineering';
```

That the query text is byte-for-byte identical across two very different engines is the whole point of a standard.

## 7. The shape of a `SELECT` (a preview)

Every query this week is a `SELECT`. Its clauses always appear in this written order:

```sql
SELECT   first_name, salary        -- which columns (projection)
FROM     employees                 -- which table
WHERE    department = 'Sales'      -- which rows (selection / filter)
ORDER BY salary DESC               -- how to sort the result
LIMIT    5;                        -- how many rows to return
```

But the database *evaluates* them in a different order — roughly `FROM` → `WHERE` → `SELECT` → `ORDER BY` → `LIMIT`. That mismatch explains a classic beginner error: you can't use a `SELECT` alias inside `WHERE`, because `WHERE` runs before `SELECT` has computed it. We'll return to this in Lecture 2; note it now.

## 8. Check yourself

- What are the everyday names for a *relation*, a *tuple*, and an *attribute*?
- Why is "a table is a set of rows" more than pedantry — what practical consequence does it have for reading query results?
- What's the difference between a primary key, a unique constraint, and a foreign key?
- In `employees`, which column is the primary key and which is a foreign key, and what does the foreign key link to?
- Name two concrete differences between PostgreSQL and SQLite, and one thing that is *identical* on both.
- What command lists tables in `psql`? In `sqlite3`?
- Why can't you reference a `SELECT` alias inside the `WHERE` clause?

If those are solid, Lecture 2 turns `SELECT` from a preview into a power tool.

## Further reading

- **PostgreSQL Tutorial — "The SQL Language":** <https://www.postgresql.org/docs/current/tutorial-sql.html>
- **SQLite — "Query Language Understood by SQLite":** <https://www.sqlite.org/lang.html>
- **Codd's original 1970 paper, "A Relational Model of Data for Large Shared Data Banks"** (a classic, and readable): <https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf>
- **Use The Index, Luke — "Preface"** (the mental model of what a database does): <https://use-the-index-luke.com/sql/preface>
