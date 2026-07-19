# Lecture 2 — Replication, read replicas & high availability

> **Duration:** ~2 hours. **Outcome:** You can explain how PostgreSQL streaming replication works from WAL to standby, set up a read replica in concept and in commands, choose synchronous vs. asynchronous commit with the durability trade-off in mind, and describe what failover is and why you orchestrate it with a tool instead of a script.

## 1. Why copy the database at all?

Partitioning made one server faster. Replication makes **more than one server** hold the data. You do it for three distinct reasons — keep them separate in your head, because they pull in different directions:

1. **Read scaling.** One primary can't serve unlimited reads. Copies (replicas) absorb read traffic.
2. **High availability (HA).** If the primary's disk dies, a replica can take over. No replica = the outage lasts as long as your restore-from-backup takes.
3. **Disaster recovery / geography.** A replica in another region survives a datacenter fire and serves nearby users with lower latency.

One mechanism — replication — serves all three, but the configuration you choose depends on which one you're buying.

## 2. The WAL is the whole story

Everything PostgreSQL replicates flows through the **Write-Ahead Log (WAL)**. You met the WAL in Week 5: before PostgreSQL changes a data page, it writes a record describing the change to the WAL and flushes it to disk. That's what makes crash recovery possible — replay the WAL and you rebuild the state.

Replication is that same idea pointed at another machine: **ship the WAL to a second server and have it replay the records.** A replica is, definitionally, a server continuously catching up by replaying the primary's WAL.

```
   Client writes
        │
        ▼
   ┌──────────┐   WAL records stream    ┌──────────┐
   │ PRIMARY  │ ──────────────────────► │ STANDBY  │
   │ (read/   │      (over TCP)         │ (read-   │
   │  write)  │                         │  only)   │
   └──────────┘                         └──────────┘
        │                                     │
     data files                          data files
                                        (rebuilt by replaying WAL)
```

## 3. Physical vs. logical replication

PostgreSQL has two native replication systems. Know both and when each fits.

| | **Physical (streaming)** | **Logical** |
|--|--------------------------|-------------|
| What ships | Raw WAL — byte-level block changes | Decoded row changes (INSERT/UPDATE/DELETE) |
| Replica is | An exact binary copy of the whole cluster | Selected tables, possibly a different schema/version |
| Replica writable? | No — read-only standby | Yes — it's a normal database receiving changes |
| Version must match? | Yes, same major version | No — can replicate across major versions |
| Typical use | Read replicas, HA, failover | Selective sync, upgrades, feeding a data warehouse |
| Set up with | `pg_basebackup` + `primary_conninfo` | `CREATE PUBLICATION` / `CREATE SUBSCRIPTION` |

**Physical/streaming** is what you reach for to build a read replica or an HA pair — it's the default meaning of "replication" in Postgres. **Logical** is for when you want *some* tables, or to move between versions, or to fan data out to other systems. This lecture focuses on physical, with a logical example at the end.

## 4. Streaming replication, step by step

Here's the shape of standing up a physical read replica. (You'll run a real version in Exercise 2 and the mini-project; this is the conceptual walkthrough.)

**On the primary** — allow replication connections and create a dedicated role:

```sql
-- A login role with the REPLICATION privilege (not a superuser).
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'strong-secret';
```

```ini
# postgresql.conf — modern defaults already allow this, but be explicit:
wal_level = replica            # minimum WAL detail needed to rebuild a standby
max_wal_senders = 10           # how many replicas/backups can stream at once
```

```conf
# pg_hba.conf — let the replicator connect to the special "replication" database.
host    replication    replicator    10.0.0.0/24    scram-sha-256
```

**Create a replication slot** so the primary keeps WAL until the replica has consumed it (otherwise a replica that falls behind can miss WAL that got recycled):

```sql
SELECT pg_create_physical_replication_slot('replica1_slot');
```

**On the standby** — take a base backup of the primary, then start up in standby mode:

```bash
# Clone the primary's data directory over the network.
pg_basebackup -h primary.internal -U replicator \
    -D /var/lib/postgresql/16/replica -R -X stream \
    -S replica1_slot -C

# -R  writes the standby connection config for you (standby.signal + primary_conninfo)
# -X stream  streams WAL during the backup so it's consistent
# -S / -C  create and use the replication slot
```

The `-R` flag writes two things into the new data directory:

- an empty **`standby.signal`** file — its mere presence tells PostgreSQL "boot as a standby, replay WAL, don't accept writes";
- **`primary_conninfo`** in `postgresql.auto.conf` — how to reach the primary.

Start the standby and it connects, streams WAL, and stays caught up. To let it also serve read queries:

```ini
hot_standby = on               # allow read-only queries on the standby (default: on)
```

**Verify it's working.** On the primary:

```sql
SELECT client_addr, state, sync_state,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS bytes_behind
FROM pg_stat_replication;
```

`state = streaming` and a small `bytes_behind` means the replica is healthy and nearly caught up.

## 5. Synchronous vs. asynchronous: the durability dial

This is the single most important trade-off in replication, and interviewers love it.

**Asynchronous (the default):** the primary commits a transaction as soon as *it* has the WAL on disk, then ships WAL to replicas in the background. Fast — the client doesn't wait on the network. The risk: if the primary dies *after* commit but *before* the replica received those last WAL records, that data is gone from the replica. You can lose the last few transactions on failover.

**Synchronous:** the primary waits to confirm the commit until at least one replica has acknowledged the WAL. Zero data loss on failover — but every commit now pays a network round-trip, and if the sync replica is down or slow, **writes stall**.

```ini
# postgresql.conf on the primary
synchronous_standby_names = 'replica1'   # names from application_name of standbys
synchronous_commit = on                  # wait for the standby to confirm
```

`synchronous_commit` has more settings than on/off — it's a spectrum of *how much* durability you buy per commit:

| Value | Primary waits until… | Trade-off |
|-------|----------------------|-----------|
| `off` | not even its own disk flush (async, local) | fastest, can lose recent local commits on crash |
| `local` | its own WAL is flushed locally | durable locally, ignores replicas |
| `on` | the sync standby has flushed WAL | zero-loss failover, slowest |
| `remote_write` | standby received WAL (not yet flushed) | middle ground |
| `remote_apply` | standby has **applied** WAL (readable there) | read-your-write on replicas, slowest |

The honest framing: **synchronous replication trades throughput and availability for durability.** A common production setup is async replicas for read scaling plus one synchronous replica for zero-loss failover, sometimes with `synchronous_standby_names = 'ANY 1 (r1, r2, r3)'` so any one of several can satisfy the quorum and one slow node doesn't stall writes.

## 6. Read replicas and the consistency you actually get

A read replica lets you route `SELECT` traffic off the primary. Great for dashboards, search, analytics, anything read-heavy. But you must understand what you're serving:

- With **async** replication, a replica is **slightly behind** the primary — usually milliseconds, sometimes seconds under load. This is **replication lag**.
- That means **reads on a replica are eventually consistent.** A user who writes a comment and immediately reads from a replica might not see their own comment yet. This is the classic **read-your-writes** problem.

Ways teams handle it:

- **Route read-after-write to the primary.** Right after a user writes, send *their* next few reads to the primary; everyone else reads replicas.
- **Wait for the LSN.** Capture the write's WAL position (LSN) and only serve that user from a replica once it has replayed past that LSN.
- **Use `remote_apply`** synchronous commit so a committed write is guaranteed visible on the replica — at a latency cost.
- **Accept the lag.** For a "trending posts" widget, three seconds stale is completely fine. Match the guarantee to the feature.

The senior instinct: **don't promise consistency the architecture can't deliver, and don't pay for consistency a feature doesn't need.**

## 7. Connection pooling — the piece people forget

Every PostgreSQL connection is a real OS process with real memory (~5–10 MB). A few hundred connections and the server spends its time context-switching, not querying. Web apps that open a connection per request blow through this fast, and adding replicas makes it worse (now you have connections to *several* servers).

A **connection pooler** sits between your app and PostgreSQL and multiplexes many client connections onto a small set of real server connections. **PgBouncer** is the standard.

```ini
# pgbouncer.ini — transaction pooling is the workhorse mode
[databases]
appdb = host=primary.internal port=5432 dbname=appdb
appdb_ro = host=replica.internal port=5432 dbname=appdb

[pgbouncer]
pool_mode = transaction        # reuse a server conn per-transaction, not per-session
max_client_conn = 5000         # clients can connect
default_pool_size = 25         # actual server connections per database
```

`pool_mode = transaction` is the high-throughput default: a server connection is handed to a client only for the duration of one transaction, then returned. The catch: **session-level features break** — session `SET`s, `LISTEN/NOTIFY`, prepared statements (pre-PG connection handling) don't survive across transactions because you're not guaranteed the same backend. Know that limitation before you turn it on.

A clean pattern: point the app at `appdb` (primary, for writes) and `appdb_ro` (replica, for reads) through PgBouncer, and PgBouncer keeps the real connection count sane on both.

## 8. High availability and failover

Read replicas scale reads. **HA** is about surviving the primary dying. The moving parts:

- **Failover:** promoting a standby to become the new primary when the old one is gone. In PostgreSQL you promote with `pg_ctl promote` (or `SELECT pg_promote();`) — the standby stops replaying, opens for writes, and becomes the primary.
- **The hard part is deciding *when*.** Automatic failover needs something to reliably detect the primary is dead and coordinate the promotion. Do it wrong and you get **split-brain**: two servers both think they're primary, both accept writes, and you now have two divergent databases to reconcile by hand. This is a genuine data-corruption incident.
- **Avoiding split-brain** needs **quorum/consensus** — an odd number of voters agreeing on who's primary, plus **fencing** the old primary so it can't accept writes if it comes back.

This is why you **do not** hand-roll failover with a bash script and a ping check. You use an orchestrator:

| Tool | What it does |
|------|--------------|
| **Patroni** | The de-facto standard. Uses a consensus store (etcd/Consul/ZooKeeper) to elect a leader, handle promotion, and fence the old primary. |
| **pg_auto_failover** | Simpler, Microsoft-backed; a monitor node arbitrates failover between primary and standby. |
| **repmgr** | Older, replication + failover management; more manual than Patroni. |
| **Managed services** | RDS, Cloud SQL, Aurora, Crunchy Bridge handle failover for you — you're paying them to run Patroni-equivalents. |

The whole HA story also needs a **virtual IP or a proxy** (HAProxy, a service-discovery layer, or the pooler) so applications reconnect to whoever is currently primary without a config change.

## 9. What about SQLite?

SQLite is a single file with no server, so "streaming replication" doesn't apply natively. The modern answers:

- **Litestream** streams SQLite's WAL to object storage (S3) continuously — effectively async replication / continuous backup for disaster recovery.
- **LiteFS** (a FUSE filesystem) replicates SQLite across nodes with one primary and read-only replicas — bringing read-replica patterns to SQLite for edge deployments.

These exist precisely because people wanted Postgres-style replicas for SQLite. For this course, know they exist; the real replication muscle is built on PostgreSQL.

## 10. Check yourself

- What physically travels from primary to standby in streaming replication?
- Give one difference between physical and logical replication, and a use case where you'd specifically pick logical.
- What does a **replication slot** protect against?
- Explain the trade-off between `synchronous_commit = on` and async replication in one sentence each.
- A user posts a comment and immediately doesn't see it. Which architecture caused this, and name two fixes.
- Why is transaction-mode pooling fast, and what does it break?
- What is split-brain, and why does it mean you shouldn't script your own failover?

If you can answer all seven, move to Lecture 3.

## Further reading

- **PostgreSQL 16 — High Availability, Load Balancing, and Replication:** <https://www.postgresql.org/docs/16/high-availability.html>
- **PostgreSQL 16 — `pg_basebackup`:** <https://www.postgresql.org/docs/16/app-pgbasebackup.html>
- **Patroni documentation:** <https://patroni.readthedocs.io/>
- **PgBouncer documentation:** <https://www.pgbouncer.org/>
