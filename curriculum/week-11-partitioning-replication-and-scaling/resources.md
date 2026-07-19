# Week 11 — Resources

Free and public unless noted. Official docs first — for scaling especially, the PostgreSQL manual is more accurate than most blog posts.

## Required reading (official PostgreSQL 16 docs)

- **Table Partitioning** — the canonical reference for declarative partitioning, pruning, and partition-wise operations:
  <https://www.postgresql.org/docs/16/ddl-partitioning.html>
  *Why:* everything in Lecture 1, straight from the source, with edge cases the lecture compresses.
- **High Availability, Load Balancing, and Replication** — the whole replication chapter:
  <https://www.postgresql.org/docs/16/high-availability.html>
  *Why:* the definitive treatment of streaming replication, standbys, and synchronous commit (Lecture 2).
- **`pg_basebackup`** — the tool you use to clone a primary into a standby:
  <https://www.postgresql.org/docs/16/app-pgbasebackup.html>
  *Why:* the exact flags used in Exercise 2 and the mini-project.
- **Logical Replication** — publications, subscriptions, and where it differs from physical:
  <https://www.postgresql.org/docs/16/logical-replication.html>
  *Why:* the "some tables / cross-version" replication you'd reach for beyond read replicas.

## The one book to buy

- **Martin Kleppmann — *Designing Data-Intensive Applications*** (O'Reilly):
  <https://dataintensive.net/>
  *Why:* chapters 5 (Replication), 6 (Partitioning), and 9 (Consistency & Consensus) are the best explanation of this entire week that exists. If you read one thing beyond the docs, read these three chapters.

## Replication & HA tooling

- **Patroni** — the de-facto HA/failover orchestrator for PostgreSQL:
  <https://patroni.readthedocs.io/>
  *Why:* how real teams get automatic, split-brain-safe failover (Lecture 2 §8).
- **PgBouncer** — the standard connection pooler:
  <https://www.pgbouncer.org/>
  *Why:* the pooling piece everyone forgets until connections thrash the primary.
- **pg_auto_failover** — a simpler failover option:
  <https://pg-auto-failover.readthedocs.io/>
  *Why:* a lighter alternative to Patroni worth knowing by name.

## Partitioning automation

- **pg_partman** — automated rolling-window partition management:
  <https://github.com/pgpartman/pg_partman>
  *Why:* the standard answer to "how do partitions get created and dropped automatically?"
- **TimescaleDB** — a PostgreSQL extension that automates time partitioning + continuous aggregates:
  <https://docs.timescale.com/>
  *Why:* if your workload is time-series, this turns Lecture 1 + rollups into a managed feature.

## Sharding & distributed PostgreSQL

- **Citus** — the open-source extension that shards PostgreSQL across nodes:
  <https://www.citusdata.com/> · <https://github.com/citusdata/citus>
  *Why:* proof that "shard Postgres" is a real, supported path (Lecture 3 §6).

## Consistency, CAP & NoSQL

- **Jepsen** — rigorous, adversarial consistency testing of real databases:
  <https://jepsen.io/analyses>
  *Why:* clarifying and sobering — shows what databases *actually* guarantee vs. what they claim.
- **"A Critique of the CAP Theorem" — Martin Kleppmann:**
  <https://arxiv.org/abs/1509.05393>
  *Why:* fixes the "pick 2 of 3" misreading; sharpens how you talk about CAP.
- **The PACELC paper (Abadi):**
  <https://en.wikipedia.org/wiki/PACELC_design_principle>
  *Why:* the extension CAP needs — latency vs. consistency during *normal* operation.

## SQLite scaling (for the comparisons)

- **Litestream** — continuous streaming replication of SQLite to object storage:
  <https://litestream.io/>
- **LiteFS** — replicated SQLite across nodes:
  <https://fly.io/docs/litefs/>
  *Why:* how the SQLite world borrows read-replica and DR patterns from Postgres.

## Glossary

| Term | Definition |
|------|------------|
| **Partition** | A physical child table holding a disjoint slice of a partitioned table's rows. |
| **Partition pruning** | The planner/executor skipping partitions a query can't need. |
| **WAL** | Write-Ahead Log — the record of every change; the substrate of both recovery and replication. |
| **Standby / replica** | A server continuously replaying the primary's WAL; read-only in physical replication. |
| **Replication slot** | State on the primary that retains WAL until a specific consumer has received it. |
| **Replication lag** | How far behind the primary a replica currently is. |
| **synchronous_commit** | How much confirmation the primary waits for before returning a commit. |
| **Failover** | Promoting a standby to primary when the old primary is gone. |
| **Split-brain** | Two servers both acting as primary — a data-corruption incident. |
| **Shard** | An independent database server holding a disjoint subset of the data. |
| **Shard key** | The column whose value (via a routing function) decides a row's home shard. |
| **CAP** | Under a network partition, choose Consistency or Availability. |
| **PACELC** | If Partition then A-vs-C; Else Latency-vs-Consistency. |
| **Eventual consistency** | Replicas converge given no new writes; reads may be temporarily stale. |
| **Polyglot persistence** | Using different data stores for the jobs each is best at. |

---

*Broken link? Open an issue or PR.*
