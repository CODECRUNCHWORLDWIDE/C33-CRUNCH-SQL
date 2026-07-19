# Exercise 1 — Partition a large table and verify pruning

**Goal:** Build a range-partitioned table, load a few million rows, and use `EXPLAIN` to *prove* that partition pruning skips the partitions your query can't need. Then break pruning on purpose so you can recognize the failure.

**Estimated time:** 60 minutes. **Stack:** PostgreSQL 16.

## Setup

Create a scratch database so you can drop it cleanly afterward:

```bash
createdb c33_w11
psql c33_w11
```

## Step 1 — Create the partitioned table

```sql
CREATE TABLE events (
    id          bigint GENERATED ALWAYS AS IDENTITY,
    created_at  timestamptz NOT NULL,
    user_id     bigint      NOT NULL,
    kind        text        NOT NULL,
    PRIMARY KEY (id, created_at)          -- PK must include the partition key
) PARTITION BY RANGE (created_at);
```

Create one partition per month for the first half of 2026, plus a default:

```sql
CREATE TABLE events_2026_01 PARTITION OF events FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE events_2026_02 PARTITION OF events FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE events_2026_03 PARTITION OF events FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
CREATE TABLE events_2026_04 PARTITION OF events FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE events_2026_05 PARTITION OF events FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE events_2026_06 PARTITION OF events FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
CREATE TABLE events_default PARTITION OF events DEFAULT;
```

Confirm the shape:

```sql
\d+ events
```

You should see the partition key and the six month-partitions listed.

## Step 2 — Load a few million rows across the months

`generate_series` makes this a one-liner. This spreads ~2.6M rows across the six months:

```sql
INSERT INTO events (created_at, user_id, kind)
SELECT ts,
       (random() * 100000)::bigint,
       (ARRAY['login','click','purchase','logout'])[1 + (random()*3)::int]
FROM generate_series('2026-01-01'::timestamptz,
                     '2026-06-30 23:00'::timestamptz,
                     '1 minute'::interval) AS g(ts);

ANALYZE events;      -- give the planner fresh statistics
```

Check the rows landed in the right partitions:

```sql
SELECT tableoid::regclass AS partition, count(*)
FROM events GROUP BY 1 ORDER BY 1;
```

Every count should sit in its own month-partition, and `events_default` should be **empty** (0 rows). If the default has rows, a `created_at` fell outside every range.

## Step 3 — Prove pruning fires

Query a single month and read the plan:

```sql
EXPLAIN
SELECT count(*) FROM events
WHERE created_at >= '2026-03-01' AND created_at < '2026-04-01';
```

The plan should reference **only `events_2026_03`** — no `Append` over every partition. That's plan-time pruning. Record the plan.

Now confirm execution-time pruning with a parameterized query:

```sql
PREPARE monthq(timestamptz, timestamptz) AS
    SELECT count(*) FROM events WHERE created_at >= $1 AND created_at < $2;

EXPLAIN (ANALYZE)
EXECUTE monthq('2026-05-01', '2026-06-01');
```

Look for a line like `Subplans Removed: N` — that's PostgreSQL pruning partitions at run time using the parameter values.

## Step 4 — Break pruning on purpose

Wrap the partition key in a function so the planner can't see the raw column:

```sql
EXPLAIN
SELECT count(*) FROM events
WHERE date_trunc('month', created_at) = '2026-03-01';
```

This plan **Appends over every partition** — pruning failed. Compare it to Step 3 and note the difference in your write-up. Then query a non-partition column:

```sql
EXPLAIN
SELECT count(*) FROM events WHERE user_id = 42;
```

Also touches every partition — `user_id` isn't the partition key, so nothing can be pruned.

## Step 5 — Feel the expiry win

```sql
-- The cheap way to drop a month: metadata-only, milliseconds.
\timing on
DROP TABLE events_2026_01;
```

Note the time. That operation is why time-series teams partition.

## Expected result

- Six populated month-partitions, an empty default.
- A Step-3 plan touching **one** partition; a Step-4 plan touching **all** of them.
- A `Subplans Removed` line proving execution-time pruning.

## Done when…

- [ ] `\d+ events` shows six range partitions plus a default.
- [ ] Rows are distributed across month-partitions; the default is empty.
- [ ] You captured a pruned plan (one partition) and an unpruned plan (all partitions) and can explain the difference.
- [ ] You saw `Subplans Removed` on the parameterized query.
- [ ] You can state in one sentence why `date_trunc('month', created_at) = ...` defeats pruning and how to rewrite it.

## Stretch

- Add a `CREATE INDEX ON events (user_id);` and re-run the `user_id = 42` query. It still touches every partition, but how does each partition's scan change?
- Turn pruning off (`SET enable_partition_pruning = off;`) and re-run Step 3. Confirm the plan now scans all partitions — proof the feature is doing the work.

## Cleanup

```sql
\q
```
```bash
dropdb c33_w11
```

Commit your plans and notes to your portfolio under `c33-week-11/exercise-01/`.
