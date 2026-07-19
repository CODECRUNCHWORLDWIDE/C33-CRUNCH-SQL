# Week 11 — Partitioning, Replication & Scaling

> *One box has a ceiling. This week you learn the three moves that push past it — split a table (partitioning), copy the database (replication), and split the data across machines (sharding) — and how to decide, honestly, when a relational database is the wrong tool at all.*

Welcome to Week 11 of **C33 · Crunch SQL**. Everything so far assumed a single PostgreSQL server that holds all your data and answers every query. That assumption breaks eventually — a table grows to a billion rows, a read workload outstrips one machine's CPU, or a write workload outstrips one machine's disk. This week is about the escape hatches, in the order you should actually reach for them: **partition first, replicate second, shard last**, and reach for NoSQL only when you can defend the trade-off out loud.

We stay grounded in real PostgreSQL 16 commands. Sharding and replication are hard to fully reproduce on a laptop, so the exercises mix commands you *run* with designs you *write and defend* — the same balance a senior engineer strikes when planning a migration they can't rehearse in production.

## Learning objectives

By the end of this week, you will be able to:

- **Partition** a large table by range, list, or hash using PostgreSQL declarative partitioning, and **prove** partition pruning happened by reading `EXPLAIN`.
- **Explain** when partitioning helps (huge tables, time-series, bulk expiry) and when it hurts (small tables, wrong partition key, cross-partition queries).
- **Describe** streaming (physical) replication end to end: WAL shipping, replication slots, hot standbys, and the synchronous-vs-asynchronous trade-off.
- **Set up** a read replica conceptually and with concrete steps, and route read traffic to it via a connection pooler.
- **Reason** about high availability and failover — what promotes a standby, what a split-brain is, and why you want a tool (Patroni, pg_auto_failover) instead of a shell script.
- **Design** a sharding key for a given workload and predict its hot spots, cross-shard queries, and rebalancing cost.
- **Make** an honest SQL-vs-NoSQL decision using CAP/PACELC and concrete consistency requirements — and defend it.

## Prerequisites

- Weeks 1–10 of C33: you can write joins, read `EXPLAIN (ANALYZE, BUFFERS)`, reason about transactions and MVCC, and pick an index.
- A local **PostgreSQL 16** install with a superuser you control (needed to edit `postgresql.conf` and create replication roles).
- **SQLite 3** for the zero-setup comparisons.
- Comfort editing config files and restarting a local service.

## Topics covered

- Declarative partitioning: `PARTITION BY RANGE | LIST | HASH`, attach/detach, the default partition
- Partition pruning at plan time vs. execution time; `enable_partition_pruning`
- Partition-wise joins and aggregates; the cost of a bad partition key
- Physical streaming replication: WAL, `pg_basebackup`, `primary_conninfo`, replication slots
- Synchronous vs. asynchronous commit; `synchronous_standby_names` and durability trade-offs
- Read replicas, read-after-write consistency, and connection pooling with PgBouncer
- HA and failover: promotion, split-brain, quorum, and orchestration tools
- Logical replication (publications/subscriptions) and where it differs from physical
- Sharding: hash vs. range shard keys, cross-shard queries, rebalancing, Citus
- CAP, PACELC, consistency models, and a defensible SQL-vs-NoSQL decision framework

## How to navigate this week

Work top to bottom. Lectures build the model; exercises make it concrete; challenges and the mini-project force the judgment calls.

| # | File | What's inside | Time |
|--:|------|---------------|-----:|
| 1 | [lecture-notes/01-partitioning.md](./lecture-notes/01-partitioning.md) | Range/list/hash partitioning, pruning, when it helps vs. hurts | 2h |
| 2 | [lecture-notes/02-replication-and-read-replicas.md](./lecture-notes/02-replication-and-read-replicas.md) | Streaming replication, read replicas, sync vs. async, HA/failover, pooling | 2h |
| 3 | [lecture-notes/03-sharding-and-sql-vs-nosql.md](./lecture-notes/03-sharding-and-sql-vs-nosql.md) | Sharding patterns + an honest SQL-vs-NoSQL decision (CAP, consistency) | 2h |
| 4 | [exercises/exercise-01-partition-and-verify-pruning.md](./exercises/exercise-01-partition-and-verify-pruning.md) | Partition a large table and prove pruning with `EXPLAIN` | 1h |
| 5 | [exercises/exercise-02-read-replica-setup.md](./exercises/exercise-02-read-replica-setup.md) | Stand up a read replica (concepts + concrete steps) | 1.25h |
| 6 | [exercises/exercise-03-design-a-sharding-key.md](./exercises/exercise-03-design-a-sharding-key.md) | Design and stress-test a sharding key for a workload | 1h |
| 7 | [challenges/challenge-01-scale-a-write-heavy-workload.md](./challenges/challenge-01-scale-a-write-heavy-workload.md) | Scale a write-heavy ingest pipeline | 1.5h |
| 8 | [challenges/challenge-02-choose-sql-vs-nosql.md](./challenges/challenge-02-choose-sql-vs-nosql.md) | Choose SQL vs. NoSQL for a case and defend it | 1.5h |
| 9 | [mini-project/README.md](./mini-project/README.md) | Partition a table + set up a read replica, documented | 4h |
| — | [homework.md](./homework.md) | Six practice problems | ~5h |
| — | [quiz.md](./quiz.md) | 14 self-check questions + answer key | 0.5h |
| — | [resources.md](./resources.md) | Curated official docs + high-quality external reading | — |

## By the end of this week you can…

- Take a table too big to manage as one heap and partition it so old data drops in milliseconds and queries touch only the partitions they need.
- Stand up a read replica, understand exactly what "eventually consistent read" means for your users, and pool connections so replicas actually absorb load.
- Sketch a sharded architecture, name its shard key, and predict where it will hurt before you build it.
- Sit in a design review and give a defensible answer to "why not just use MongoDB / DynamoDB / Cassandra?" — grounded in CAP, consistency requirements, and the actual access pattern.

## A note on honesty

Scaling advice is full of cargo-culting. The most senior move in this whole week is knowing that **most databases never need any of this** — a well-indexed single Postgres box on modern hardware serves enormous workloads. Reach for partitioning when a table is genuinely huge, replication when reads or availability demand it, and sharding only when you've exhausted the first two. Premature sharding has killed more projects than slow queries ever did.

## Up next

[Week 12 — Capstone: Tune & Design](../week-12-capstone-tune-and-design/) — put all twelve weeks together and drive a real database to a latency target.

---

*Part of Code Crunch Worldwide · GPL-3.0 · If you find errors, open an issue or PR.*
