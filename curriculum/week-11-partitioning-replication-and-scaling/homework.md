# Week 11 — Homework

Six problems, ~5 hours total. Commit each to your portfolio under `c33-week-11/homework/`. These reinforce the lectures and exercises — do them after the exercises, before the quiz.

---

## Problem 1 — List partitioning (40 min)

Exercise 1 used range partitioning. Now do **list** partitioning.

Create a `customers` table partitioned by `LIST (region)` with partitions for `'us'`, `'eu'`, `'apac'`, and a default. Insert a handful of rows across the regions, plus one row with region `'latam'` (which has no partition).

**Acceptance (`list-partitioning.md`):**

- The DDL for the parent and all partitions.
- Proof (a `SELECT tableoid::regclass, ...`) that the `'latam'` row landed in the **default** partition.
- An `EXPLAIN` for `WHERE region = 'eu'` showing only `customers_eu` is scanned.
- One sentence: when is LIST the right choice over RANGE or HASH?

---

## Problem 2 — Hash partitioning + distribution (40 min)

Create a table partitioned by `HASH (user_id)` into 4 partitions. Load 100,000 rows with random `user_id`s.

**Acceptance (`hash-partitioning.md`):**

- The `CREATE TABLE ... PARTITION BY HASH` DDL and the four `FOR VALUES WITH (MODULUS 4, REMAINDER n)` partitions.
- A `SELECT tableoid::regclass, count(*)` showing rows spread roughly evenly across the four.
- One sentence explaining why a `WHERE user_id = 500` query touches only one partition, but `WHERE user_id BETWEEN 1 AND 1000` touches all four.

---

## Problem 3 — Read the replication docs and diagram it (45 min)

From memory first, then checking the docs, draw the data flow of physical streaming replication: client write → WAL on primary → WAL sender → WAL receiver → replay on standby → readable.

**Acceptance (`replication-diagram.md` + an image or Mermaid):**

- A diagram labeling: WAL, `wal_sender`, `wal_receiver`, replication slot, `hot_standby`.
- A caption explaining what a **replication slot** guarantees and what breaks without one.
- Two sentences contrasting physical vs. logical replication.

---

## Problem 4 — The synchronous-commit trade-off (45 min)

Without necessarily running it, reason through the five `synchronous_commit` levels from Lecture 2 §5.

**Acceptance (`sync-commit.md`):**

- A table of the five levels (`off`, `local`, `on`, `remote_write`, `remote_apply`) with, for each: what the primary waits for, and the failure scenario in which you'd lose data.
- A recommendation for each of these three systems, with justification:
  1. A bank ledger.
  2. A high-traffic analytics event pipeline.
  3. A dashboard that must show a user's own write immediately from a replica.

---

## Problem 5 — Connection pooling math (30 min)

Your web app runs 50 app-server instances, each with a connection pool of 20 → up to 1,000 connections hitting Postgres. The primary starts thrashing above ~200 connections.

**Acceptance (`pooling.md`):**

- Explain, with numbers, why 1,000 direct connections is a problem (per-connection memory, context switching).
- A PgBouncer config sketch (`pool_mode`, `max_client_conn`, `default_pool_size`) that lets 1,000 clients share ~40 real server connections.
- One sentence on what `pool_mode = transaction` breaks and one app feature you'd have to avoid because of it.

---

## Problem 6 — Sharding trade-off essay (40 min)

In `sharding-essay.md` (300–400 words), argue **both sides** of: *"We should shard our database now."* Your team's main table is 400 GB on a box that's 40% utilized, growing 5%/month.

**Acceptance:**

- The case *for* sharding (steelman it).
- The case *against* (the honest one, given those numbers).
- Your verdict, with the specific signal that *would* make you change it.
- Name one thing sharding would break that currently "just works" (a join, a constraint, a transaction).

---

## Time budget

| Problem | Time |
|--------:|-----:|
| 1 | 40 min |
| 2 | 40 min |
| 3 | 45 min |
| 4 | 45 min |
| 5 | 30 min |
| 6 | 40 min |
| **Total** | **~4.5 h** |

After homework, take the [quiz](./quiz.md), then build the [mini-project](./mini-project/README.md).
