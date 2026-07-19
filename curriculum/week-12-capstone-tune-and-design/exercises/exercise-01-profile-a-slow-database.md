# Exercise 1 — Profile a slow database

**Goal:** Stand up a realistic, deliberately slow database, generate a workload, and use `pg_stat_statements` + `EXPLAIN (ANALYZE, BUFFERS)` to name its three worst queries. You will *not* fix anything yet — this rep is about measuring.

**Estimated time:** ~2 hours.

## Step 0 — Enable pg_stat_statements

In `postgresql.conf` (find it with `SHOW config_file;` in `psql`):

```conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
```

Restart Postgres (`pg_ctl restart`, `brew services restart postgresql@16`, or `sudo systemctl restart postgresql` — whichever your install uses).

## Step 1 — Build the seed database

```bash
createdb crunch_tune
psql crunch_tune
```

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- A shop schema with NO indexes beyond primary keys, on purpose.
CREATE TABLE customers (
    id      bigserial PRIMARY KEY,
    email   text NOT NULL,
    country text NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE orders (
    id          bigserial PRIMARY KEY,
    customer_id bigint NOT NULL REFERENCES customers(id),
    status      text NOT NULL,          -- 'paid','pending','refunded','cancelled'
    total       numeric(12,2) NOT NULL,
    created_at  timestamptz NOT NULL
);

-- 200k customers
INSERT INTO customers (email, country, created_at)
SELECT 'user' || g || '@example.com',
       (ARRAY['US','GB','DE','FR','BR','IN'])[1 + (g % 6)],
       now() - (random() * interval '900 days')
FROM generate_series(1, 200000) g;

-- 5,000,000 orders spread over ~2 years
INSERT INTO orders (customer_id, status, total, created_at)
SELECT 1 + (random() * 199999)::bigint,
       (ARRAY['paid','paid','paid','pending','refunded','cancelled'])[1 + (floor(random()*6))::int],
       round((random() * 500)::numeric, 2),
       now() - (random() * interval '730 days')
FROM generate_series(1, 5000000) g;

ANALYZE;   -- give the planner fresh stats
```

This takes a minute or two. You now have 5M orders with only the primary-key indexes — the foreign key `orders.customer_id` is deliberately **not** indexed (Postgres does not auto-index FKs), and there is no index on `status` or `created_at`.

## Step 2 — Generate a workload

Run each of these several times (they mimic a dashboard + checkout). Reset the stats first so you measure a clean window:

```sql
SELECT pg_stat_statements_reset();
```

```sql
-- Q1: recent high-value orders (dashboard)
SELECT o.id, o.total, c.email
FROM orders o JOIN customers c ON c.id = o.customer_id
WHERE o.created_at >= now() - interval '7 days'
ORDER BY o.total DESC LIMIT 20;

-- Q2: one customer's order history (account page)
SELECT id, status, total, created_at
FROM orders WHERE customer_id = 12345 ORDER BY created_at DESC;

-- Q3: revenue by country (report)
SELECT c.country, count(*) AS orders, sum(o.total) AS revenue
FROM orders o JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'paid'
GROUP BY c.country ORDER BY revenue DESC;

-- Q4: refund count for a customer (support tool)
SELECT count(*) FROM orders WHERE customer_id = 6789 AND status = 'refunded';
```

Run the block 5–10 times (up-arrow in `psql`, or wrap in a `\i workload.sql` file and `\i` it repeatedly) so the counters accumulate realistic call counts.

## Step 3 — Find where the time is

```sql
SELECT substr(query, 1, 55) AS query,
       calls,
       round(total_exec_time::numeric, 1) AS total_ms,
       round(mean_exec_time::numeric, 2)  AS mean_ms,
       round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 1) AS pct
FROM pg_stat_statements
WHERE query LIKE '%orders%'
ORDER BY total_exec_time DESC
LIMIT 10;
```

Record the top three in `report.md`. For each, note whether it looks like "genuinely slow" (high mean) or "fast but frequent" (high calls, low mean).

## Step 4 — Read each offender's plan

For your top three, capture the plan:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, status, total, created_at
FROM orders WHERE customer_id = 12345 ORDER BY created_at DESC;
```

For each plan, write one sentence naming the bottleneck using the Lecture-1 vocabulary: `Seq Scan`, `Rows Removed by Filter`, a bad estimate, a spilling `Sort`, or a large `Buffers: read`.

## Expected result

- Q2 and Q4 show a **`Seq Scan on orders`** scanning all 5M rows to find one customer's handful — because `customer_id` has no index. `Rows Removed by Filter` ≈ 5,000,000.
- Q1 shows a seq scan + sort of everything in the last 7 days.
- Q3 aggregates the whole table (a seq scan is arguably fine here — hold that thought for Exercise 2).

## Done when…

- [ ] `crunch_tune` exists with 200k customers and 5M orders.
- [ ] `pg_stat_statements` returns your workload's queries, sorted by `total_exec_time`.
- [ ] `report.md` names the **three** worst queries with their `total_ms`, `mean_ms`, and `calls`.
- [ ] Each of the three has a captured `EXPLAIN (ANALYZE, BUFFERS)` plan and a one-sentence diagnosis.
- [ ] You changed **nothing** yet. Measuring only.

## Stretch

- Run the same four queries in a SQLite build of this schema (`.timer on`, `EXPLAIN QUERY PLAN`) and compare how SQLite describes the same scans.
- Add `pg_stat_statements` columns `shared_blks_read` and `shared_blks_hit` to your query — which offender does the most disk I/O?

## Submission

Commit `report.md` to your portfolio under `c33-week-12/exercise-01/`.
