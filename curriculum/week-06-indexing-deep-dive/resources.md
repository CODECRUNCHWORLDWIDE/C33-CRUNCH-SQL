# Week 6 — Resources

Free, public, no signup. The PostgreSQL manual is the primary source all week; the rest are the clearest secondary explanations of indexing that exist.

## Primary reference — the PostgreSQL manual

- **Indexes (chapter overview)** — start here; short and readable:
  <https://www.postgresql.org/docs/current/indexes.html>
- **Index Types** — btree/hash/GIN/GiST/SP-GiST/BRIN and their operators:
  <https://www.postgresql.org/docs/current/indexes-types.html>
- **Multicolumn Indexes** — the leftmost-prefix rule, in the source:
  <https://www.postgresql.org/docs/current/indexes-multicolumn.html>
- **Partial Indexes** — with the canonical examples:
  <https://www.postgresql.org/docs/current/indexes-partial.html>
- **Index-Only Scans and Covering Indexes** — the visibility-map caveat explained:
  <https://www.postgresql.org/docs/current/indexes-index-only-scans.html>
- **`CREATE INDEX`** — full syntax: `USING`, `INCLUDE`, `CONCURRENTLY`, opclasses:
  <https://www.postgresql.org/docs/current/sql-createindex.html>
- **Using EXPLAIN** — how to read the planner's output, with worked examples:
  <https://www.postgresql.org/docs/current/using-explain.html>
- **Row Estimation Examples** — how selectivity is estimated from statistics:
  <https://www.postgresql.org/docs/current/row-estimation-examples.html>

## Index types in depth

- **GIN:** <https://www.postgresql.org/docs/current/gin.html> — why it's an inverted index and how `fastupdate` works.
- **GiST:** <https://www.postgresql.org/docs/current/gist.html> — the bounding-box framework behind ranges, geometry, KNN.
- **BRIN:** <https://www.postgresql.org/docs/current/brin.html> — block-range summaries; note the correlation requirement.
- **JSON Indexing:** <https://www.postgresql.org/docs/current/datatype-json.html#JSON-INDEXING> — `jsonb_ops` vs `jsonb_path_ops`.
- **`pg_trgm`:** <https://www.postgresql.org/docs/current/pgtrgm.html> — trigram indexes for `LIKE '%x%'` and similarity.

## The best external explainer

- **"Use The Index, Luke!" — Markus Winand** (free web book; the single best resource on how indexing *actually* works, engine-agnostic with a Postgres slant):
  <https://use-the-index-luke.com/>
  - Anatomy of an index: <https://use-the-index-luke.com/sql/anatomy>
  - The `WHERE` clause (sargability, column order): <https://use-the-index-luke.com/sql/where-clause>
  - Sorting & grouping with indexes: <https://use-the-index-luke.com/sql/sorting-grouping>

## Reading and visualizing plans

- **`EXPLAIN` glossary — depesz** — what every plan node and field means:
  <https://www.depesz.com/tag/explain/>
- **explain.depesz.com** — paste a plan, get it color-coded by where the time goes (free):
  <https://explain.depesz.com/>
- **explain.dalibo.com (PEV2)** — a visual plan tree, great for spotting the expensive node:
  <https://explain.dalibo.com/>

## Statistics, bloat, and maintenance

- **`pg_stats` view** — the statistics the planner uses (`n_distinct`, MCVs, correlation):
  <https://www.postgresql.org/docs/current/view-pg-stats.html>
- **`pg_stat_user_indexes`** — `idx_scan` to find unused indexes:
  <https://www.postgresql.org/docs/current/monitoring-stats.html>
- **Routine Vacuuming** — why `VACUUM` keeps index-only scans fast and indexes un-bloated:
  <https://www.postgresql.org/docs/current/routine-vacuuming.html>
- **`pgstattuple` / `pageinspect`** — measure bloat and peek inside index pages:
  <https://www.postgresql.org/docs/current/pgstattuple.html>

## SQLite contrast (for the curious)

- **SQLite Query Optimizer Overview** — how a much smaller engine makes similar choices:
  <https://www.sqlite.org/optoverview.html>
- **SQLite `EXPLAIN QUERY PLAN`:** <https://www.sqlite.org/eqp.html>

## Glossary

| Term | Definition |
|------|------------|
| **Selectivity** | Fraction of rows a predicate keeps. Low fraction → index wins; high fraction → seq scan wins. |
| **Sargable** | A predicate written so an index can be used — the column is bare, not wrapped in a function. |
| **Heap** | The table's row storage. An index entry points into the heap via a `ctid`. |
| **Heap fetch** | Following an index entry to read the actual row from the table — random I/O. |
| **Index-only scan** | Answering a query from the index alone, no heap fetch — the fastest read. |
| **Covering index** | An index (often via `INCLUDE`) that contains every column a query needs. |
| **Composite index** | A multi-column index; usable only via a leftmost prefix of its columns. |
| **Partial index** | An index with a `WHERE` clause, covering only a subset of rows. |
| **Bitmap Heap Scan** | Index builds a page bitmap, then reads heap pages in physical order — for medium selectivity. |
| **Visibility map** | Per-page bitmap of all-visible pages; lets index-only scans skip the heap. Refreshed by `VACUUM`. |
| **Bloat** | Dead index/heap entries left by MVCC until `VACUUM` reclaims them. |

---

*Broken link? Open an issue or PR.*
