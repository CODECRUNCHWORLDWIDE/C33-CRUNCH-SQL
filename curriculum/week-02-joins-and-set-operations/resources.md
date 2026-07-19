# Week 2 — Resources

Free, public, no signup unless noted. Each link has a one-line "why this."

## Official documentation (the source of truth)

- **PostgreSQL 16 — Table Expressions / Joined Tables** — the normative reference
  for every join type and `ON`/`USING`/`NATURAL`:
  <https://www.postgresql.org/docs/16/queries-table-expressions.html#QUERIES-JOIN>
  *Why:* precise, complete, and it's the engine you're running.
- **PostgreSQL 16 — Combining Queries (`UNION`/`INTERSECT`/`EXCEPT`)**:
  <https://www.postgresql.org/docs/16/queries-union.html>
  *Why:* the set-operation rules, including `ALL` semantics, straight from the docs.
- **PostgreSQL 16 — Subquery Expressions (`EXISTS`, `IN`, `NOT IN`)**:
  <https://www.postgresql.org/docs/16/functions-subquery.html>
  *Why:* the exact semantics behind semi-joins and anti-joins.
- **SQLite — `SELECT` (joins + compound selects)**:
  <https://www.sqlite.org/lang_select.html>
  *Why:* confirms SQLite's join support and the 3.39 `RIGHT`/`FULL` note.

## Learn / practice interactively

- **PostgreSQL Tutorial — Joins section** (free, worked examples):
  <https://www.postgresqltutorial.com/postgresql-tutorial/postgresql-joins/>
  *Why:* clean diagrams and runnable examples for each join type.
- **SQLBolt — interactive lessons 6–9 (multi-table joins)**:
  <https://sqlbolt.com/lesson/select_queries_with_joins>
  *Why:* in-browser, immediate feedback, exactly this week's scope.
- **pgexercises — the "Joins and subqueries" set** (free, gradeable):
  <https://pgexercises.com/questions/joins/>
  *Why:* dozens of real join puzzles with hints and answers, on Postgres.

## Visualize joins

- **"A Visual Explanation of SQL Joins" — Jeff Atwood**:
  <https://blog.codinghorror.com/a-visual-explanation-of-sql-joins/>
  *Why:* the classic Venn-diagram mental model (read it, but remember joins aren't
  *really* set intersections — that caveat is on the page).
- **Join diagram + live SQL (joins.spathon.com)**:
  <https://joins.spathon.com/>
  *Why:* pick a join, see the rows it keeps, with the SQL beside it.

## Going deeper (optional, high quality)

- **Markus Winand — "Use The Index, Luke": the JOIN operation**:
  <https://use-the-index-luke.com/sql/join>
  *Why:* connects joins to how the engine executes them — a bridge to Week 7.
- **Modern SQL — set operations across databases**:
  <https://modern-sql.com/caniuse/intersect>
  *Why:* shows which engines support `INTERSECT`/`EXCEPT ALL`, useful for portability.
- **"Don't Do This" — PostgreSQL wiki (`NOT IN` with NULL, and more)**:
  <https://wiki.postgresql.org/wiki/Don%27t_Do_This>
  *Why:* the canonical warning behind this week's `NOT EXISTS`-over-`NOT IN` rule.

## Reference cards

- **PostgreSQL `psql` cheat sheet** (the `\d`, `\dt`, `\x` meta-commands you'll
  use to inspect the schema): <https://www.postgresql.org/docs/16/app-psql.html>
- **SQLite CLI dot-commands** (`.tables`, `.schema`, `.mode column`, `.headers on`):
  <https://www.sqlite.org/cli.html>

## Glossary

| Term | Definition |
|------|------------|
| **Join** | Combine rows from two tables side by side, matched by a condition. |
| **Cartesian product** | Every row of A paired with every row of B; `CROSS JOIN`. |
| **Inner join** | Keep only matching pairs; unmatched rows dropped. |
| **Outer join** | Keep unmatched rows from one (`LEFT`/`RIGHT`) or both (`FULL`) sides, filling `NULL`. |
| **Self-join** | A table joined to itself, distinguished by two aliases. |
| **Anti-join** | Rows of A with **no** match in B (`NOT EXISTS` / `LEFT JOIN … IS NULL`). |
| **Semi-join** | Rows of A **with** a match in B, without multiplying (`EXISTS` / `IN`). |
| **Grain** | What one row of a result represents (an order? a line item?). |
| **Junction table** | A table (like `order_items`) that links two others in a many-to-many. |
| **`USING`** | Join shorthand when both tables share the key's name; merges the column. |
| **Set operation** | Combine two result sets as sets: `UNION`, `INTERSECT`, `EXCEPT`. |
| **Union-compatible** | Two queries with the same column count and compatible types. |

---

*Broken link? Open an issue or PR.*
