# Week 8 — Resources

Free, public, no signup. Official docs first, then the deep dives that actually made window functions click for people.

## Primary references (bookmark these)

- **PostgreSQL 16 — Window Functions tutorial** — the gentlest official intro, with the `empsalary` example: <https://www.postgresql.org/docs/16/tutorial-window.html>
  *Why: the canonical "aha" walkthrough of `OVER`/`PARTITION BY` from the source of truth.*
- **PostgreSQL 16 — Window Function Calls (syntax of frames)** — the exact grammar for `ROWS`/`RANGE`/`GROUPS`, `EXCLUDE`, and the `WINDOW` clause: <https://www.postgresql.org/docs/16/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS>
  *Why: when a frame does something surprising, the answer is on this page.*
- **PostgreSQL 16 — Built-in window functions** (`row_number`, `rank`, `lag`, `lead`, `ntile`, …): <https://www.postgresql.org/docs/16/functions-window.html>
  *Why: the one-line semantics of every ranking/offset function, including default arguments.*
- **SQLite — Window Functions** — thorough and readable, with a worked frame section: <https://www.sqlite.org/windowfunctions.html>
  *Why: your zero-setup engine; also one of the clearest frame explainers anywhere.*

## Setup (if you haven't yet)

- **Install PostgreSQL 16** — official downloads for macOS/Linux/Windows: <https://www.postgresql.org/download/>
  *Why: the primary engine for this course; window support is complete.*
- **Postgres.app (macOS, one-click)**: <https://postgresapp.com/>
  *Why: fastest way onto Postgres on a Mac, no Homebrew needed.*
- **SQLite CLI** — ships on macOS/Linux; Windows binaries here: <https://www.sqlite.org/download.html>
  *Why: `sqlite3 file.db` and you're querying in five seconds — great for the exercises.*
- **DB Fiddle** — run Postgres/SQLite in the browser to share a reproducible query: <https://www.db-fiddle.com/>
  *Why: paste a seed + query and get a shareable link when you want help.*

## Deep dives worth your time

- **"Window Functions in SQL" — Mode SQL Tutorial** (free, interactive): <https://mode.com/sql-tutorial/sql-window-functions/>
  *Why: the best plain-English narrative of `PARTITION BY` vs `GROUP BY` with runnable examples.*
- **Use The Index, Luke — "Window Functions" chapter** by Markus Winand: <https://use-the-index-luke.com/sql/partial-results/window-functions>
  *Why: connects window functions to indexing and performance — the Week-7 angle on Week-8 queries.*
- **"The SQL of Gaps and Islands in Sequences"** (engine-agnostic, Itzik Ben-Gan's method): <https://www.red-gate.com/simple-talk/databases/sql-server/t-sql-programming-sql-server/the-sql-of-gaps-and-islands-in-sequences/>
  *Why: the definitive treatment of the pattern behind Exercise 3 and Challenge 2.*
- **"How to Analyze Cohort Retention with SQL"** — a practical cohort-triangle walkthrough: <https://www.metabase.com/learn/metabase-basics/querying-and-dashboards/questions/cohort-analysis>
  *Why: the exact chart Challenge 1 and the mini-project build, explained by an analytics team.*

## Practice grounds

- **PostgreSQL Exercises** — free, graded, includes a window-function section: <https://pgexercises.com/questions/aggregates/>
  *Why: extra reps with instant feedback, on a schema you don't have to build.*
- **SQL Murder Mystery** — puzzle-based SQL practice (uses joins + aggregation heavily): <https://mystery.knightlab.com/>
  *Why: keeps drilling fun once the mechanics are in place.*

## Reference card

| Task | Pattern |
|------|---------|
| Rank within group | `ROW_NUMBER()/RANK()/DENSE_RANK() OVER (PARTITION BY g ORDER BY x DESC)` |
| Top-N per group | rank in a CTE, then `WHERE rn <= N` outside |
| Running total | `SUM(x) OVER (ORDER BY t ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` |
| Moving average (7-row) | `AVG(x) OVER (ORDER BY t ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)` |
| Period-over-period | `x - LAG(x) OVER (ORDER BY t)` |
| First / last in group | `FIRST_VALUE(x)` / `LAST_VALUE(x) OVER (… ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)` |
| Percent of total | `x * 100.0 / SUM(x) OVER ()` |
| Islands (runs) | group by `x - ROW_NUMBER() OVER (ORDER BY x)` |
| Gaps | `LEAD(x) OVER (ORDER BY x) - x > 1` |
| Sessionize | `LAG` gap → `CASE` flag → running `SUM(flag)` |
| Pivot | `SUM(x) FILTER (WHERE k = 'v')` (or `SUM(CASE WHEN … END)`) |

---

*Broken link? Open an issue or PR. Part of the Code Crunch Worldwide open curriculum · GPL-3.0.*
