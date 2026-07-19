# Week 9 — Resources

Free, official-first. Read the PostgreSQL docs before third-party blogs — for server-side logic they are unusually clear and always current.

## Official PostgreSQL 16 docs (the primary source)

- **`CREATE VIEW`** — syntax, updatable views, `WITH CHECK OPTION`. Why: the definitive reference for what a view can and can't do.
  <https://www.postgresql.org/docs/16/sql-createview.html>
- **`CREATE MATERIALIZED VIEW`** — creation, `WITH [NO] DATA`. Why: shows the storage-vs-freshness trade at the syntax level.
  <https://www.postgresql.org/docs/16/sql-creatematerializedview.html>
- **`REFRESH MATERIALIZED VIEW`** — the `CONCURRENTLY` rules and the unique-index requirement. Why: the one page that explains why your concurrent refresh errors out.
  <https://www.postgresql.org/docs/16/sql-refreshmaterializedview.html>
- **PL/pgSQL — full chapter** — variables, control flow, `RETURN QUERY`, error trapping. Why: the single most important read of the week.
  <https://www.postgresql.org/docs/16/plpgsql.html>
- **`CREATE FUNCTION`** — languages, argument modes, `RETURNS TABLE`, volatility. Why: the reference you'll reopen every time you write a function signature.
  <https://www.postgresql.org/docs/16/sql-createfunction.html>
- **Function volatility categories** — `IMMUTABLE`/`STABLE`/`VOLATILE`. Why: gets a whole page because getting it wrong is a real correctness bug.
  <https://www.postgresql.org/docs/16/xfunc-volatility.html>
- **Trigger behavior overview** — timing, level, firing order. Why: the mental model for `BEFORE`/`AFTER` × `ROW`/`STATEMENT`.
  <https://www.postgresql.org/docs/16/trigger-definition.html>
- **`CREATE TRIGGER`** — full syntax including transition tables and `WHEN`. Why: for the bulk-path and no-op-guard patterns.
  <https://www.postgresql.org/docs/16/sql-createtrigger.html>
- **Triggers in PL/pgSQL** — `NEW`/`OLD`/`TG_OP` and the return-value contract. Why: the exact rules the exercises depend on.
  <https://www.postgresql.org/docs/16/plpgsql-trigger.html>
- **Generated columns** — the `STORED` derived-column feature. Why: the tool that replaces most derived-column triggers.
  <https://www.postgresql.org/docs/16/ddl-generated-columns.html>
- **`CREATE PROCEDURE` / transaction control** — when you need `COMMIT` mid-body. Why: the function-vs-procedure boundary.
  <https://www.postgresql.org/docs/16/sql-createprocedure.html>

## Extensions worth knowing

- **`pg_cron`** — schedule `REFRESH` and maintenance jobs from inside the database. Why: the simplest materialized-view refresh scheduler (Challenge 1).
  <https://github.com/citusdata/pg_cron>
- **`btree_gist`** — enables exclusion constraints over ranges + equality. Why: the native "no overlap" enforcement in Challenge 2.
  <https://www.postgresql.org/docs/16/btree-gist.html>

## Deeper reads and patterns

- **PostgreSQL wiki — Audit trigger** — a battle-tested generic audit-trigger implementation. Why: compare it to the one you wrote; it handles edge cases you may have missed.
  <https://wiki.postgresql.org/wiki/Audit_trigger_91plus>
- **"Don't do this" (PostgreSQL wiki)** — community list of anti-patterns, several about triggers and functions. Why: sharpens the "when NOT to" judgement from Lecture 3.
  <https://wiki.postgresql.org/wiki/Don%27t_Do_This>
- **Range types** — `tstzrange`, the `&&` overlap operator. Why: underpins the exclusion-constraint approach to the booking rule.
  <https://www.postgresql.org/docs/16/rangetypes.html>

## SQLite (for contrast)

- **SQLite — `CREATE TRIGGER`** — what SQLite triggers can and can't do vs. PostgreSQL. Why: know the boundary before you carry these patterns to a SQLite project.
  <https://www.sqlite.org/lang_createtrigger.html>
- **SQLite — `CREATE VIEW`** — views yes, materialized views no. Why: confirms why the materialization work in this week is PostgreSQL-only.
  <https://www.sqlite.org/lang_createview.html>

## Practice environments

- **Free local Postgres** — install via your OS package manager, Docker (`postgres:16`), or Postgres.app on macOS. Why: you want a throwaway database to break; don't practice triggers on anything real.
- **DB Fiddle (PostgreSQL 16)** — run snippets in the browser, share a link. Why: quick to test a trigger without local setup.
  <https://www.db-fiddle.com/>

---

*Broken link or a better resource? Open an issue or PR.*
