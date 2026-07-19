# Week 11 — Quiz

Fourteen questions. Lectures closed. Aim for 12/14 before starting Week 12.

---

**Q1.** In PostgreSQL declarative partitioning, the **parent** table:

- A) Holds all the rows; partitions are just indexes.
- B) Holds no rows itself — it routes rows to partitions.
- C) Must be dropped before you can query the partitions.
- D) Can only have hash partitions.

---

**Q2.** Which partition strategy is the natural fit for time-series data you expire by age?

- A) LIST
- B) HASH
- C) RANGE
- D) None — you can't partition time-series.

---

**Q3.** A primary key on a partitioned table must:

- A) Be a single column.
- B) Include the partition key column(s).
- C) Be a UUID.
- D) Live only on the default partition.

---

**Q4.** This query fails to prune partitions on a table partitioned by `RANGE (created_at)`:

```sql
WHERE date_trunc('month', created_at) = '2026-03-01'
```

Why?

- A) `date_trunc` isn't a real function.
- B) The partition key is wrapped in a function, so the planner can't map the predicate to a range.
- C) Pruning only works on HASH partitions.
- D) You must disable `enable_partition_pruning` first.

---

**Q5.** Which `EXPLAIN` output signals **execution-time** partition pruning?

- A) `Seq Scan on events`
- B) `Append`
- C) `Subplans Removed: N`
- D) `Bitmap Heap Scan`

---

**Q6.** The cheap way to delete a whole month from a range-partitioned table is:

- A) `DELETE FROM events WHERE created_at < ...`
- B) `TRUNCATE events`
- C) `DROP TABLE events_2026_01` (or `DETACH PARTITION`)
- D) `VACUUM FULL events`

---

**Q7.** In physical streaming replication, what actually travels from primary to standby?

- A) Decoded row-level INSERT/UPDATE/DELETE statements.
- B) The raw WAL (write-ahead log records).
- C) Full table snapshots every second.
- D) SQL query text.

---

**Q8.** A **replication slot** on the primary primarily prevents:

- A) The standby from accepting writes.
- B) The primary from recycling WAL a lagging standby hasn't consumed yet.
- C) Split-brain during failover.
- D) The need for `pg_basebackup`.

---

**Q9.** With `synchronous_commit = on` and one synchronous standby, if that standby goes down:

- A) Nothing changes; writes continue at full speed.
- B) The primary keeps committing but silently loses durability.
- C) Commits on the primary block until a synchronous standby is available.
- D) The primary automatically demotes itself.

---

**Q10.** A user posts a comment, then immediately reads from an **async** read replica and doesn't see it. This is:

- A) A bug in PostgreSQL.
- B) The read-your-writes problem caused by replication lag.
- C) Impossible — replicas are always current.
- D) A sign the replica has crashed.

---

**Q11.** PgBouncer's `pool_mode = transaction` is fast because it reuses a server connection per-transaction, but it **breaks**:

- A) All SELECT queries.
- B) Session-level state like session `SET`s and `LISTEN/NOTIFY`.
- C) Primary keys.
- D) Replication.

---

**Q12.** **Split-brain** is:

- A) A replica that's slightly behind the primary.
- B) Two servers both believing they're primary and both accepting writes.
- C) A partitioned table with two default partitions.
- D) A query that scans two partitions.

---

**Q13.** What does sharding scale that read replicas cannot?

- A) Read throughput.
- B) Write throughput / total data size beyond one primary.
- C) Backup speed.
- D) Index size.

---

**Q14.** The CAP theorem says that **during a network partition**, a distributed system must choose between:

- A) Cost and Performance.
- B) Consistency and Availability.
- C) Latency and Throughput.
- D) SQL and NoSQL.

---

## Answer key

<details>
<summary>Reveal after attempting</summary>

1. **B** — the parent is a router; it stores no rows.
2. **C** — RANGE on the time column; expire by dropping old partitions.
3. **B** — uniqueness can only be enforced within a partition, so the PK must include the partition key.
4. **B** — wrapping `created_at` in `date_trunc` hides the raw column; filter on the bare column with `>=`/`<` instead.
5. **C** — `Subplans Removed: N` shows partitions pruned at execution time (e.g. from parameters).
6. **C** — `DROP TABLE` (or `DETACH PARTITION`) is a metadata operation; the `DELETE` scans and bloats.
7. **B** — raw WAL records. (Logical replication ships decoded row changes — a different system.)
8. **B** — the slot makes the primary retain WAL until the standby has consumed it, preventing gaps.
9. **C** — writes block until a synchronous standby acknowledges. That's the availability cost of zero-loss durability. (A common mitigation: `ANY 1 (r1, r2, r3)` so any one of several satisfies the quorum.)
10. **B** — replication lag on an async replica; classic read-your-writes. Fix by routing read-after-write to the primary or waiting for the LSN.
11. **B** — transaction pooling doesn't guarantee the same backend across statements, so session-level state doesn't persist.
12. **B** — two primaries accepting writes = divergent data; why you use quorum-based orchestration (Patroni) rather than a script.
13. **B** — replicas scale reads; sharding scales writes and total data across servers.
14. **B** — Consistency vs. Availability, *when a partition occurs*. PACELC adds latency-vs-consistency during normal operation.

</details>

Scored 12+? Move to the mini-project. 9–11: re-read the lecture behind each miss. Below 9: re-read all three lectures before Week 12.
