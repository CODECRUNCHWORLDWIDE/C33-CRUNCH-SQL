# Week 12 — Resources

Free, public, no signup unless noted. Curated for tuning, plan-reading, and schema design — the three skills the capstone demands.

## Required reading (official PostgreSQL 16 docs)

- **Using EXPLAIN** — how to read the plan tree, node by node. The single most important page this week:
  <https://www.postgresql.org/docs/16/using-explain.html>
- **pg_stat_statements** — the per-query telemetry that tells you *what* to tune:
  <https://www.postgresql.org/docs/16/pgstatstatements.html>
- **Planner cost constants & statistics** — why the planner picks a plan, and how stats drive it:
  <https://www.postgresql.org/docs/16/runtime-config-query.html>
- **Routine Vacuuming** — MVCC bloat, autovacuum, and why a query slows over months with no code change:
  <https://www.postgresql.org/docs/16/routine-vacuuming.html>

## Plan-reading tools (keep open in tabs)

- **explain.dalibo.com** — paste an `EXPLAIN (ANALYZE, BUFFERS)` plan, get a visual tree that highlights the expensive nodes and bad estimates. Indispensable for the capstone:
  <https://explain.dalibo.com/>
- **PEV2 (pgMustard's explain visualizer, free tier)** — annotated plans with hints on what to fix:
  <https://www.pgmustard.com/> (the free explain visualizer; the paid product is optional)

## Tuning references

- **Use The Index, Luke! — Markus Winand** — the best free book on how indexes and the SQL you write interact. Read "Anatomy of an Index" and "The Where Clause" this week:
  <https://use-the-index-luke.com/>
- **PostgreSQL Wiki — Slow Query Questions** — the checklist experts ask before helping you; a great self-review:
  <https://wiki.postgresql.org/wiki/Slow_Query_Questions>
- **PostgreSQL Wiki — Don't Do This** — a curated list of type and schema anti-patterns straight from the community:
  <https://wiki.postgresql.org/wiki/Don%27t_Do_This>

## Schema design & anti-patterns

- **SQL Antipatterns — Bill Karwin** — the canonical catalog (EAV, polymorphic associations, and more). Not free, but the chapter list itself is a review checklist:
  <https://pragprog.com/titles/bksqla/sql-antipatterns/>
- **PostgreSQL — Data Types** — pick the narrowest correct type; the cheapest constraint you have:
  <https://www.postgresql.org/docs/16/datatype.html>
- **PostgreSQL — JSON Types** — when `jsonb` + GIN is the right answer instead of EAV:
  <https://www.postgresql.org/docs/16/datatype-json.html>

## Benchmarking & monitoring

- **pgbench** — the built-in benchmarking tool for repeatable p95 numbers:
  <https://www.postgresql.org/docs/16/pgbench.html>
- **auto_explain** — log the plan of any slow query automatically, in production:
  <https://www.postgresql.org/docs/16/auto-explain.html>
- **postgres_exporter (Prometheus)** — ship `pg_stat_*` to Grafana for trends and alerts:
  <https://github.com/prometheus-community/postgres_exporter>

## SQLite (zero-setup track)

- **EXPLAIN QUERY PLAN** — SQLite's plan reader; the method mirrors Postgres:
  <https://www.sqlite.org/eqp.html>
- **The Query Optimizer Overview** — how SQLite chooses indexes and join order:
  <https://www.sqlite.org/optoverview.html>

## Videos & talks (free)

- **"How PostgreSQL's planner uses statistics"** and similar talks from PGConf are on YouTube — search the official **PostgreSQL** and **PGConf** channels for EXPLAIN and query-tuning talks:
  <https://www.youtube.com/@PostgresOpen>

## Glossary

| Term | Definition |
|------|------------|
| **p95 / p99** | The latency the slowest 5% / 1% of requests still beat. What users feel. |
| **Seq Scan** | Read the whole table. Fine for small tables; a smell on a big table with a selective filter. |
| **Buffers** | 8 KB pages read (`read`) or found in cache (`hit`). High `read` = disk-bound. |
| **Selectivity** | The fraction of rows a predicate keeps. Low selectivity (few rows) favors an index. |
| **Covering index** | An index holding every column a query needs, enabling an `Index Only Scan`. |
| **Bloat** | Dead tuples left by MVCC that autovacuum hasn't reclaimed; wastes space and cache. |
| **EAV** | Entity–attribute–value: storing fields as rows. Flexible, but slow and type-unsafe. |
| **N+1** | Fetch a list (1 query), then one query per item (N). Replace with a single join. |
| **Latency budget** | The target a query must meet (e.g. p95 < 100 ms). Tuning stops here, not at "fastest possible." |
| **Working set** | The hot rows + indexes actually touched by the workload. Performance holds while it fits in RAM. |

---

*Broken link? Open an issue. Part of Code Crunch Worldwide · GPL-3.0.*
