# Week 3 — Resources

Free, official-first. Read the PostgreSQL docs for the authoritative behaviour; use the others to build intuition.

## Official documentation (the source of truth)

- **PostgreSQL 16 — Aggregate Functions:** <https://www.postgresql.org/docs/16/functions-aggregate.html>
  *Why:* the canonical list of `COUNT`/`SUM`/`AVG`/`MIN`/`MAX`, plus `FILTER` and ordered-set aggregates. Bookmark it.
- **PostgreSQL 16 — GROUP BY / GROUPING SETS / ROLLUP / CUBE:** <https://www.postgresql.org/docs/16/queries-table-expressions.html#QUERIES-GROUPING-SETS>
  *Why:* the only precise reference for subtotal semantics and the `GROUPING()` marker.
- **PostgreSQL 16 — Subquery Expressions (`IN`, `ANY`, `ALL`, `EXISTS`):** <https://www.postgresql.org/docs/16/functions-subquery.html>
  *Why:* nails down the NULL behaviour of `NOT IN` vs `NOT EXISTS` — the trap from Lecture 2.
- **PostgreSQL 16 — WITH Queries (CTEs + RECURSIVE + CYCLE):** <https://www.postgresql.org/docs/16/queries-with.html>
  *Why:* the recursive-CTE execution model and the `MATERIALIZED`/`CYCLE` clauses, straight from the source.
- **SQLite — Aggregate Functions:** <https://www.sqlite.org/lang_aggfunc.html>
  *Why:* confirms which aggregates (and `FILTER`, 3.30+) SQLite supports so your queries stay portable.
- **SQLite — The WITH Clause (recursive):** <https://www.sqlite.org/lang_with.html>
  *Why:* SQLite's recursive-CTE reference, including its lack of `GROUPING SETS`/`CYCLE`.

## Interactive practice

- **PostgreSQL Exercises — Aggregation section:** <https://pgexercises.com/questions/aggregates/>
  *Why:* graded, in-browser aggregation problems against a realistic schema; closest thing to this week's exercises online.
- **SQLZoo — SUM and COUNT / SELECT within SELECT:** <https://sqlzoo.net/wiki/SUM_and_COUNT> and <https://sqlzoo.net/wiki/SELECT_within_SELECT_Tutorial>
  *Why:* bite-sized subquery and aggregation drills with instant feedback.
- **DB Fiddle:** <https://www.db-fiddle.com/>
  *Why:* paste the `crunch_shop` seed and share a query by URL — great for asking for help or checking Postgres-vs-SQLite differences.

## Explainers worth your time

- **Use The Index, Luke — "The GROUP BY clause":** <https://use-the-index-luke.com/sql/sorting-grouping>
  *Why:* connects grouping to how the engine actually executes it — a bridge to Week 6–7.
- **Modern SQL — "GROUPING SETS, ROLLUP, CUBE":** <https://modern-sql.com/feature/grouping-sets>
  *Why:* the clearest cross-database walkthrough of subtotals, with a support matrix per engine.
- **Modern SQL — "WITH (Common Table Expressions)" and recursion:** <https://modern-sql.com/feature/with>
  *Why:* excellent visual explanation of how a recursive CTE iterates.

## Reference card

| Concept | Remember |
|---------|----------|
| `COUNT(*)` vs `COUNT(col)` | `*` counts rows; `col` skips NULLs. Difference = number of NULLs. |
| `WHERE` vs `HAVING` | `WHERE` filters rows before grouping (no aggregates); `HAVING` filters groups after (aggregates OK). |
| `FILTER (WHERE …)` | Per-aggregate row filter. Postgres + SQLite 3.30+. Fallback: `SUM(CASE …)`. |
| `ROLLUP`/`CUBE`/`GROUPING SETS` | Subtotals. **Postgres only** (not SQLite). NULL marks a subtotal; `GROUPING()` confirms it. |
| `NOT IN` + NULL | Returns zero rows if the subquery has a NULL. Prefer `NOT EXISTS`. |
| Correlated subquery | References an outer column; conceptually runs per outer row. |
| CTE | Named temp result for one statement. Chain with commas; later reads earlier. |
| `WITH RECURSIVE` | anchor `UNION ALL` recursive-term. **Always** add a loop guard (depth cap / `CYCLE`). |

## Glossary

| Term | Definition |
|------|------------|
| **Aggregate function** | A function that reduces a set of rows to one value (`SUM`, `COUNT`, …). |
| **Group** | A bucket of rows sharing the same `GROUP BY` key. |
| **Grouping set** | One specific combination of columns to group by; `ROLLUP`/`CUBE` are shorthands for many. |
| **Scalar subquery** | A subquery returning exactly one row and column, usable as a value. |
| **Correlated subquery** | A subquery that references the outer query and re-evaluates per outer row. |
| **Semi-join** | The plan the engine uses for `IN`/`EXISTS`: "keep outer rows that have a match." |
| **CTE** | Common Table Expression — a named `WITH` block scoped to one statement. |
| **Optimisation fence** | A boundary the planner won't optimise across; pre-PG12 CTEs were one. |
| **Recursive CTE** | A `WITH RECURSIVE` that references itself to walk a hierarchy of unknown depth. |

---

*Broken link? Open an issue.*
