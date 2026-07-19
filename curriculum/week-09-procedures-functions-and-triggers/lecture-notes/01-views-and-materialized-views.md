# Lecture 1 — Views and Materialized Views

> **Duration:** ~2 hours. **Outcome:** You can create a view to name a complex query, create a materialized view to cache expensive results, refresh it (with and without `CONCURRENTLY`), and choose between the two by read/write ratio and freshness tolerance.

By now you have written queries that are three CTEs deep with two window functions and a `HAVING` clause. They work. But you paste them from a scratch file every time, and when the schema changes you have to find every copy. A **view** fixes that: it gives a query a name and stores the *definition* in the database. A **materialized view** goes further — it stores the *result*, trading freshness for speed. This lecture is about both, and about the judgement call between them.

## 1. Why views exist

A view is a named query. Nothing more, nothing less. When you `SELECT` from a view, PostgreSQL substitutes the view's definition into your query and runs the combined thing. The view holds **no data of its own** — it is a saved `SELECT`.

Three reasons to reach for one:

1. **Naming.** `SELECT * FROM active_customers` reads better than 15 lines of join-and-filter, and everyone on the team means the same thing by "active customer."
2. **Abstraction.** You can reshape the schema underneath a view without breaking the queries that read it — the view is a stable contract.
3. **Security.** You can grant access to a view that exposes three columns of a ten-column table, and withhold access to the base table. (Week 10 builds on this.)

The cost is close to zero: a view adds no storage and, because the planner inlines it, usually no measurable overhead.

## 2. Creating and using a view

Syntax is `CREATE VIEW name AS <query>`:

```sql
CREATE VIEW active_customers AS
SELECT c.customer_id,
       c.full_name,
       c.email,
       c.country
FROM   customers c
WHERE  c.status = 'active'
  AND  c.deleted_at IS NULL;
```

Now query it like a table:

```sql
SELECT country, count(*)
FROM   active_customers
GROUP BY country
ORDER BY count(*) DESC;
```

Views compose. You can build a view on top of a view, join a view to a table, and put a view in a CTE. Use `CREATE OR REPLACE VIEW` to update a definition without dropping it — but note you can only *add* trailing columns, not remove or reorder existing ones, without a `DROP VIEW ... CASCADE` first.

Inspect a view's definition any time:

```sql
\d+ active_customers          -- in psql, shows the definition
SELECT pg_get_viewdef('active_customers', true);   -- pretty-printed SQL
```

## 3. Updatable views and `WITH CHECK OPTION`

A simple view (one table, no aggregates, no `DISTINCT`, no `GROUP BY`) is **updatable** — you can `INSERT`/`UPDATE`/`DELETE` through it and PostgreSQL applies the change to the base table.

```sql
UPDATE active_customers SET country = 'CA' WHERE customer_id = 42;
```

The danger: nothing stops you from updating a row *out of* the view's own filter. `WITH CHECK OPTION` forbids that:

```sql
CREATE VIEW active_customers AS
SELECT customer_id, full_name, email, country, status
FROM   customers
WHERE  status = 'active'
WITH CHECK OPTION;

-- This now FAILS, because it would move the row out of the view:
UPDATE active_customers SET status = 'closed' WHERE customer_id = 42;
```

For anything more complex than a single-table view, use an `INSTEAD OF` trigger (Lecture 3) to define exactly what a write means.

## 4. The problem views do NOT solve: cost

A view re-runs its query *every time you select from it*. If the underlying query aggregates ten million rows and takes eight seconds, then every dashboard hit takes eight seconds. A view saved you typing; it saved you nothing at runtime.

This is where **materialized views** come in.

## 5. Materialized views: cache the result

A materialized view runs its query **once**, at creation or refresh time, and stores the resulting rows on disk like a real table. Reads are then as fast as reading a table — because it *is* a table, plus a saved definition of how to rebuild it.

```sql
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT date_trunc('month', o.placed_at) AS month,
       o.country,
       count(*)                          AS order_count,
       sum(o.total_cents) / 100.0        AS revenue
FROM   orders o
WHERE  o.status = 'paid'
GROUP BY 1, 2
WITH DATA;
```

`WITH DATA` (the default) populates it immediately. `WITH NO DATA` creates it empty and unqueryable until the first refresh — useful when you want to create the object now and populate it later in a maintenance window.

The trade you just made: **reads got fast, but the data is now a snapshot.** New paid orders do not appear in `monthly_revenue` until you refresh it.

## 6. Refreshing

You rebuild the snapshot with `REFRESH`:

```sql
REFRESH MATERIALIZED VIEW monthly_revenue;
```

This re-runs the query and replaces the stored rows. The catch: a plain `REFRESH` takes an **`ACCESS EXCLUSIVE` lock** for its duration — every reader blocks until it finishes. For a big report on a busy dashboard, that is a visible outage.

### `REFRESH ... CONCURRENTLY`

`CONCURRENTLY` rebuilds the data without blocking readers — they keep seeing the old snapshot until the new one is ready, then swap:

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;
```

Two requirements and one cost:

- **Requirement:** the materialized view must have a **`UNIQUE` index** covering a set of columns that uniquely identifies every row. PostgreSQL uses it to compute the diff.
- **Requirement:** it cannot be run inside a larger transaction block the way a plain refresh can.
- **Cost:** it is *slower* overall than a plain refresh (it computes and applies a delta), but it does not block reads.

Create the required unique index once:

```sql
CREATE UNIQUE INDEX monthly_revenue_uq
    ON monthly_revenue (month, country);
```

Now `REFRESH ... CONCURRENTLY` works.

## 7. View vs. materialized view — the decision

| Question | View | Materialized view |
|----------|------|-------------------|
| Stores data? | No (definition only) | Yes (result on disk) |
| Data freshness | Always current | As of last refresh |
| Read cost | Runs the full query each time | Reads stored rows (fast) |
| Write cost | None | `REFRESH` re-computes |
| Can you index it? | No (index the base tables) | Yes — it's real storage |
| Blocks on refresh? | N/A | Plain: yes. `CONCURRENTLY`: no |
| Best when | Query is cheap, or freshness is mandatory | Query is expensive, and stale-by-minutes is OK |

The heuristic in one sentence: **use a materialized view when the query is expensive, it is read far more often than the data changes, and your users can tolerate the data being a few minutes (or hours) old.** A revenue dashboard refreshed every 15 minutes is a perfect fit. A bank balance is not — never materialize something that must be exactly current.

## 8. Refresh strategies (preview of Challenge 1)

You have three broad options for *when* to refresh, and this is a real design decision:

| Strategy | How | Trade-off |
|----------|-----|-----------|
| **Scheduled** | `cron` / `pg_cron` runs `REFRESH ... CONCURRENTLY` every N minutes | Simple; data is up to N minutes stale |
| **On-demand** | App triggers a refresh after a known bulk load | Fresh right after loads; needs a hook |
| **Trigger-driven** | A trigger on the base table queues/forces a refresh | Near-real-time; can crush write throughput — usually a mistake |

For most reporting workloads, **scheduled** wins on simplicity. `pg_cron` (an extension) keeps the schedule inside the database:

```sql
-- Refresh every 15 minutes (requires the pg_cron extension)
SELECT cron.schedule('refresh-monthly-revenue',
                      '*/15 * * * *',
                      'REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue');
```

Challenge 1 asks you to reason through the choice for a specific workload — don't just default to trigger-driven because it sounds "real-time."

## 9. SQLite note

SQLite has **views** (`CREATE VIEW`) but **no materialized views** and no `REFRESH`. The usual SQLite substitute is a plain table you `INSERT INTO ... SELECT` and rebuild yourself, or a view plus application-side caching. If you are prototyping in SQLite, build the view; do the materialization work in PostgreSQL.

## 10. Housekeeping

```sql
DROP VIEW IF EXISTS active_customers;
DROP MATERIALIZED VIEW IF EXISTS monthly_revenue;

-- See all materialized views and whether they've ever been populated:
SELECT matviewname, ispopulated FROM pg_matviews;
```

An unpopulated materialized view (`ispopulated = false`, from `WITH NO DATA`) will error if you query it — refresh it first.

## 11. Check yourself

- What does a view store? What does a materialized view store?
- Why does selecting from a view over a 10-second query take 10 seconds?
- What lock does a plain `REFRESH MATERIALIZED VIEW` take, and who does it block?
- What are the two requirements for `REFRESH ... CONCURRENTLY`?
- Give one thing you should *never* put in a materialized view.
- Which refresh strategy is usually the wrong default, and why?

When all six are easy, do [Exercise 1](../exercises/exercise-01-view-and-sql-function.md).

## Further reading

- **PostgreSQL — `CREATE VIEW`:** <https://www.postgresql.org/docs/16/sql-createview.html>
- **PostgreSQL — `CREATE MATERIALIZED VIEW`:** <https://www.postgresql.org/docs/16/sql-creatematerializedview.html>
- **PostgreSQL — `REFRESH MATERIALIZED VIEW`:** <https://www.postgresql.org/docs/16/sql-refreshmaterializedview.html>
