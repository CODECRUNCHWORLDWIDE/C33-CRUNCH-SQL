# Lecture 1 — Partitioning: splitting one table into many

> **Duration:** ~2 hours. **Outcome:** You can partition a table by range, list, or hash in PostgreSQL 16, explain what partition pruning does and prove it happened by reading `EXPLAIN`, and decide — with reasons — whether a given table should be partitioned at all.

## 1. The problem partitioning solves

A single PostgreSQL table is one logical thing but many physical files on disk (a "heap" plus its indexes). As it grows to hundreds of millions or billions of rows, three specific pains show up:

1. **Maintenance gets expensive.** `VACUUM`, `ANALYZE`, and `REINDEX` run over the whole table. Deleting last year's data means a giant `DELETE` that bloats the table and thrashes the WAL.
2. **Indexes get huge.** A B-tree over a billion rows is deep and cache-unfriendly, even when the query only cares about last week.
3. **Queries scan more than they need.** A dashboard for "this month" still has to navigate an index built over all of history.

**Partitioning** splits one large table into many smaller physical tables (called *partitions*) that together behave like the original. You still `INSERT` into and `SELECT` from one table name; PostgreSQL routes rows to the right partition and — crucially — skips partitions a query can't possibly need.

The mental model: a partitioned table is a **router**, not a storage table. It holds no rows itself. Each partition is a real table holding a disjoint slice of the data.

## 2. The three partition strategies

PostgreSQL supports three built-in **declarative** partitioning methods. You choose one at table-creation time based on how you query and expire the data.

| Method | Splits by | Best for | Example key |
|--------|-----------|----------|-------------|
| **RANGE** | a value falling in a `[from, to)` interval | time-series, anything with a natural ordering you filter on | `created_at` by month |
| **LIST** | a value being in an explicit set | discrete categories you query one at a time | `region` in `('us','eu','apac')` |
| **HASH** | `hash(key) mod N` | spreading rows evenly when there's no natural range/list | `customer_id` across 8 partitions |

Rule of thumb: **RANGE** for time (by far the most common), **LIST** for a small fixed set of categories, **HASH** when you just want even distribution and don't query by ranges of the key.

## 3. Range partitioning, hands-on

Say you have an `events` table that grows forever. Partition it by month on `created_at`.

```sql
-- The parent: declares the shape and the partition key. Holds NO data.
CREATE TABLE events (
    id          bigint GENERATED ALWAYS AS IDENTITY,
    created_at  timestamptz NOT NULL,
    user_id     bigint      NOT NULL,
    kind        text        NOT NULL,
    payload     jsonb
) PARTITION BY RANGE (created_at);

-- One partition per month. Bounds are [FROM inclusive, TO exclusive).
CREATE TABLE events_2026_01 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE events_2026_02 PARTITION OF events
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

CREATE TABLE events_2026_03 PARTITION OF events
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
```

Now insert without caring which partition a row lands in:

```sql
INSERT INTO events (created_at, user_id, kind)
VALUES ('2026-02-14 09:30+00', 42, 'login');
-- PostgreSQL routes this to events_2026_02 automatically.
```

**The primary-key gotcha.** A primary key or unique constraint on a partitioned table *must include the partition key*. This is a hard rule: PostgreSQL can only guarantee uniqueness within a partition, so the key has to be part of what's unique.

```sql
-- WRONG: id alone can't be the PK on a partitioned table.
-- ERROR: unique constraint on partitioned table must include partition key column
ALTER TABLE events ADD PRIMARY KEY (id);

-- RIGHT: include the partition key.
ALTER TABLE events ADD PRIMARY KEY (id, created_at);
```

### The DEFAULT partition

Insert a row for a month with no partition and you get an error. A **default partition** catches anything that doesn't fit:

```sql
CREATE TABLE events_default PARTITION OF events DEFAULT;
```

Useful as a safety net, but treat it as a bug alarm, not a plan — rows piling up in `events_default` mean you forgot to create a partition. It also *slows down* adding new partitions later (PostgreSQL must scan the default to check no rows belong in the new range).

## 4. The killer feature: dropping and detaching data

The reason time-series shops love partitioning: **expiring old data is instant.**

```sql
-- Deleting a month of data the old way: slow, bloats the table, floods the WAL.
DELETE FROM events WHERE created_at < '2026-01-01';   -- scans + rewrites

-- With partitions: a metadata operation. Milliseconds.
DROP TABLE events_2026_01;

-- Or keep it around, just remove it from the live set (e.g. to archive):
ALTER TABLE events DETACH PARTITION events_2026_01;
```

`DETACH ... CONCURRENTLY` (PostgreSQL 14+) removes a partition without holding a heavy lock, so live traffic keeps flowing. This — cheap bulk expiry — is often the single biggest win, bigger than the query speedups.

## 5. Partition pruning: the query-time payoff

**Partition pruning** is PostgreSQL deciding, for a given query, which partitions it can skip entirely. If a query filters `WHERE created_at >= '2026-02-01' AND created_at < '2026-03-01'`, only `events_2026_02` can possibly match — the others are never touched.

There are two flavors, and knowing which one fired matters:

| Kind | When it happens | Triggered by |
|------|-----------------|--------------|
| **Plan-time pruning** | while planning, from constant/literal predicates | `WHERE created_at >= '2026-02-01'` |
| **Execution-time pruning** | while running, from values not known until execution | parameters (`$1`), subquery results, nested-loop join keys |

Pruning is controlled by `enable_partition_pruning` (on by default). Prove it with `EXPLAIN`:

```sql
EXPLAIN
SELECT count(*) FROM events
WHERE created_at >= '2026-02-01' AND created_at < '2026-03-01';
```

A pruned plan mentions **only** the surviving partition:

```
Aggregate
  ->  Seq Scan on events_2026_02 events
        Filter: (created_at >= '2026-02-01' ... )
```

An **unpruned** plan lists every partition under an `Append` node — a red flag that your predicate doesn't match the partition key, or the key is wrapped in a function the planner can't see through:

```
Aggregate
  ->  Append
        ->  Seq Scan on events_2026_01
        ->  Seq Scan on events_2026_02
        ->  Seq Scan on events_2026_03
```

For execution-time pruning (parameterized queries), `EXPLAIN (ANALYZE)` shows lines like `Subplans Removed: 2`, telling you two partitions were skipped at run time.

### What breaks pruning

- **Wrapping the key in a function.** `WHERE date_trunc('month', created_at) = '2026-02-01'` hides the raw column; the planner can't prune. Filter on the bare column instead: `created_at >= '2026-02-01' AND created_at < '2026-03-01'`.
- **Querying on a non-partition column.** `WHERE user_id = 42` prunes nothing — every partition might hold that user. You'll scan them all (though each partition's own index still helps).
- **Type mismatches** that force an implicit cast the planner won't push down.

## 6. Partition-wise joins and aggregates

When you join two tables partitioned the **same way** on the **same key**, PostgreSQL can join matching partitions pairwise instead of joining the whole tables — a *partition-wise join*. Same idea for grouped aggregates. These are off by default because they cost planning time; enable them when you have many partitions:

```sql
SET enable_partitionwise_join = on;
SET enable_partitionwise_aggregate = on;
```

This only pays off when both tables share the partition key and boundaries. It's an advanced knob — know it exists, reach for it when a big partitioned join is slow.

## 7. Indexes on partitioned tables

Create the index on the **parent** and PostgreSQL creates a matching index on every partition, now and in the future:

```sql
CREATE INDEX ON events (user_id);          -- propagates to all partitions
CREATE INDEX ON events (created_at, kind);
```

Each partition's index is independent and smaller, which is part of the maintenance win. You can also index a single partition directly for a partition-specific access pattern.

## 8. When partitioning HELPS vs. HURTS

Partitioning is not free and not always a win. Be honest about it.

| Partitioning HELPS when… | Partitioning HURTS when… |
|--------------------------|--------------------------|
| The table is genuinely huge (100M+ rows, or growing without bound) | The table is small — you added complexity for nothing |
| You expire data in bulk by time (drop old partitions) | You never delete old data and never query by the key |
| Most queries filter on the partition key (pruning fires) | Queries filter on *other* columns, so every partition is scanned anyway |
| You have a natural range/list/hash key | There's no clean key; you're forcing one |
| Maintenance (VACUUM/REINDEX) on the whole table is painful | You have thousands of tiny partitions — planning overhead dominates |

Two failure modes worth naming:

- **Too few partitions:** partitions still huge, little benefit. **Too many partitions** (thousands): planning slows because the planner considers each one; queries that can't prune pay a per-partition tax. Aim for tens to low hundreds of partitions, not thousands.
- **Wrong key:** if you partition by month but always query by `user_id`, you get all the overhead and none of the pruning. Partition by how you *query*, not by what feels tidy.

## 9. What about SQLite?

SQLite has **no native partitioning** — no `PARTITION BY`. If you need something like it, the common workarounds are:

- **Separate tables + a `UNION ALL` view** (`events_2026_01`, `events_2026_02`, …) — you route inserts in application code and query the view. This is essentially manual partitioning and PostgreSQL's declarative feature exists precisely to replace it.
- **`ATTACH DATABASE`** to spread data across multiple files, one per slice.

For most SQLite use (embedded, single-file, moderate size) you don't need this — it's a signal you may have outgrown SQLite. Partitioning at scale is a PostgreSQL job.

## 10. Automating partition creation

Nobody wants to hand-create a partition every month. In production you either:

- run a scheduled job (cron / `pg_cron`) that creates next month's partition ahead of time, or
- use the **`pg_partman`** extension, which manages a rolling window of partitions (create ahead, drop/detach behind) for you.

You don't need `pg_partman` for this course, but know the name — "how do partitions get created automatically?" is a real interview question and the answer is "a scheduled job or pg_partman," not "PostgreSQL does it magically."

## 11. Check yourself

- What does the **parent** table in a partitioned setup actually store? (Nothing — it's a router.)
- Why must a primary key on a partitioned table include the partition key?
- Write a `WHERE` clause that prunes on `created_at`, and one that *fails* to prune because it wraps the column in a function.
- What's the difference between plan-time and execution-time pruning, and which `EXPLAIN` line reveals the latter?
- Give one workload where partitioning helps a lot and one where it only adds overhead.
- Why is `DROP TABLE events_2026_01` so much cheaper than `DELETE FROM events WHERE ...`?

If you can answer all six without scrolling up, you're ready for Lecture 2.

## Further reading

- **PostgreSQL 16 docs — Table Partitioning:** <https://www.postgresql.org/docs/16/ddl-partitioning.html>
- **PostgreSQL 16 docs — Partition Pruning:** <https://www.postgresql.org/docs/16/ddl-partitioning.html#DDL-PARTITION-PRUNING>
- **pg_partman:** <https://github.com/pgpartman/pg_partman>
