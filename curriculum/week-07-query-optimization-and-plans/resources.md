# Week 7 — Resources

Free, public, no signup unless noted. Curated — every link earns its place.

## Required reading

- **PostgreSQL — Using EXPLAIN** — the canonical walkthrough of plan output; read it twice:
  <https://www.postgresql.org/docs/current/using-explain.html>
- **PostgreSQL — Planner statistics and how they're used** — where the row estimates come from:
  <https://www.postgresql.org/docs/current/planner-stats.html>
- **PostgreSQL — Query planning configuration** — the cost constants and `work_mem`, all in one page:
  <https://www.postgresql.org/docs/current/runtime-config-query.html>

## Plan-reading tools (keep these bookmarked)

- **explain.dalibo.com** — paste a `FORMAT JSON` plan, get an interactive visual tree that highlights the expensive nodes and bad estimates. The best free plan visualizer: <https://explain.dalibo.com/>
- **explain.depesz.com** — the classic; colour-codes each node by its share of total time and flags estimate errors: <https://explain.depesz.com/>
- **pgMustard** — commercial but has a free tier and excellent *explanations* of what to fix (not just what's slow): <https://www.pgmustard.com/>

## Deep dives worth your time

- **Use The Index, Luke!** — Markus Winand's free book on indexing and the planner; the sargability and "why isn't my index used" chapters are essential this week: <https://use-the-index-luke.com/>
- **PostgreSQL — Row estimation examples** — worked arithmetic of how selectivity is computed from `pg_stats`: <https://www.postgresql.org/docs/current/row-estimation-examples.html>
- **PostgreSQL — Multivariate (extended) statistics examples** — the `CREATE STATISTICS` fix for correlated columns, with numbers: <https://www.postgresql.org/docs/current/multivariate-statistics-examples.html>
- **PostgreSQL Wiki — Slow Query Questions** — the checklist experienced people ask for; internalize it and you'll debug faster: <https://wiki.postgresql.org/wiki/Slow_Query_Questions>

## Join and scan internals

- **PostgreSQL — Planner/optimizer overview** — how candidate plans are generated and costed: <https://www.postgresql.org/docs/current/planner-optimizer.html>
- **The Internals of PostgreSQL (Suzuki)** — free online book; the "Query Processing" chapter diagrams nested loop / hash / merge joins beautifully: <https://www.interdb.jp/pg/pgsql03.html>

## SQLite side

- **SQLite — EXPLAIN QUERY PLAN** — how to read SQLite's leaner plan output: <https://www.sqlite.org/eqp.html>
- **SQLite — The Query Optimizer Overview** — how SQLite chooses indexes and why `ANALYZE` matters: <https://www.sqlite.org/optoverview.html>
- **SQLite — The `ANALYZE` command** — populates `sqlite_stat1`; run it after bulk loads: <https://www.sqlite.org/lang_analyze.html>

## Configuration helpers

- **PGTune** — generates sane `postgresql.conf` values (including `work_mem`, `effective_cache_size`) for your RAM and workload. A starting point, not gospel — always measure: <https://pgtune.leopard.in.ua/>

## Video (free)

- **CMU Intro to Database Systems (Andy Pavlo)** — the query optimization and cost-model lectures are graduate-quality and free: <https://15445.courses.cs.cmu.edu/>

## Glossary

| Term | Definition |
|------|------------|
| **Plan node** | One operation in the execution tree (Seq Scan, Hash Join, Sort, …). |
| **Startup cost** | Estimated cost before the first row is emitted. High for sorts. |
| **Total cost** | Estimated cost to emit the last row. The number the planner minimizes. |
| **Selectivity** | Fraction of rows a predicate is estimated to keep. Drives scan/join choice. |
| **Sargable** | A predicate that can use an index (the indexed column is left bare). |
| **Seq Scan** | Read every page of a table in physical order. |
| **Index Only Scan** | Answer from the index alone, no heap fetch (needs a covering index + fresh visibility map). |
| **Bitmap Heap Scan** | Read matching pages once, in physical order, from a bitmap of index matches. |
| **Nested Loop / Hash / Merge Join** | The three join algorithms. |
| **`work_mem`** | Memory a sort or hash node may use before spilling to disk. Per-node, per-connection. |
| **ANALYZE (statement)** | Collects table statistics for the planner. Distinct from the EXPLAIN option. |
| **Extended statistics** | `CREATE STATISTICS` — teaches the planner about correlated columns. |
| **`pg_stats`** | The view exposing per-column statistics (`n_distinct`, MCVs, histogram, `null_frac`). |

---

*Broken link? Open an issue.*
