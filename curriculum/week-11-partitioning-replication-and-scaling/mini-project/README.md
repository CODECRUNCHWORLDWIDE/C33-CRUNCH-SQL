# Mini-Project — Partition a table + set up a read replica

> Take a realistic table, partition it by time, load it, prove pruning works — then stand up a physical read replica of the whole database, route a read to it, and document the entire thing so someone else could reproduce it from your notes alone.

**Estimated time:** 4 hours, spread across the back half of the week. **Stack:** PostgreSQL 16.

This mini-project ties the week together: partitioning (Lecture 1 / Exercise 1) *and* replication (Lecture 2 / Exercise 2) in one coherent, documented build. The deliverable is not just a working setup — it's a **runbook** clear enough that a teammate could rebuild it without asking you a single question. Documentation is the skill being graded as much as the SQL.

---

## Deliverable

A directory in your portfolio `c33-week-11/mini-project/` containing:

1. `runbook.md` — the step-by-step build, with every command and its purpose. The centerpiece.
2. `schema.sql` — the partitioned table DDL and the automated partition-creation function.
3. `evidence.md` — pasted `EXPLAIN` plans, `pg_stat_replication` output, and lag measurements that *prove* it works.
4. `reflection.md` — 300–400 words on what you'd do differently for a production version.

---

## Part A — Partition a realistic table (≈1.5h)

Build an `orders` table partitioned by month on `order_date`, holding a few million rows.

**Requirements:**

- Range-partition on `order_date`, one partition per month, covering at least 6 months.
- A primary key that includes the partition key.
- At least one secondary index created on the parent (so it propagates to all partitions).
- A **default partition** as a safety net.
- Load ≥ 3 million rows spread across the months (use `generate_series`).
- **Automated partition creation:** write a PL/pgSQL function `create_month_partition(target date)` that creates next month's partition if it doesn't exist. Show it being called. (This is the "how do partitions get made automatically" answer — a scheduled call to this function.)

**Prove it:**

- Capture an `EXPLAIN` plan for a single-month query showing **only one partition** touched.
- Capture a plan for a query on a non-partition column showing **all** partitions touched — and explain the difference in `evidence.md`.
- Time a `DROP TABLE orders_<oldest_month>` and note how fast bulk expiry is.

## Part B — Set up a read replica (≈1.5h)

Stand up a physical streaming replica of the database (two clusters on one machine, as in Exercise 2).

**Requirements:**

- Primary configured for replication (`wal_level = replica`, a replication role, a physical slot).
- Replica cloned with `pg_basebackup -R`, started in `hot_standby` mode on its own port.
- Confirm streaming via `pg_stat_replication` on the primary.
- Demonstrate that the partitioned `orders` table — including partitions created *after* the replica started — appears on the replica. (Partition DDL replicates too; verify it.)
- Show the replica rejecting a write (read-only error).
- Measure replication lag under a bulk insert and show it recovering.

**Prove it:** paste the `pg_stat_replication` row, the read-only error, and before/after lag numbers into `evidence.md`.

## Part C — Tie them together (≈0.5h)

- Write on the primary, read the same data back from the replica — through the partitioned table — and confirm pruning still works *on the replica* (`EXPLAIN` on the replica). Partitioning and replication compose; show it.
- In `reflection.md`, answer: for a real deployment, would these reads tolerate replica lag? Which queries would you pin to the primary and why?

---

## Acceptance criteria

- [ ] `schema.sql` creates a correctly partitioned `orders` table with an automated partition-creation function.
- [ ] `evidence.md` shows a pruned plan (one partition) and an unpruned plan (all partitions), each labeled.
- [ ] A read replica is streaming; `pg_stat_replication` output is captured.
- [ ] The replica mirrors partitions created after it started, and rejects writes.
- [ ] Replication lag was measured under load and shown recovering.
- [ ] `runbook.md` is complete enough to reproduce the whole build from scratch — a teammate could follow it blind.

---

## Rubric

| Criterion | Weight | "Great" looks like |
|-----------|------:|--------------------|
| Partitioning correctness | 25% | Right key, PK includes it, automated creation, default partition |
| Pruning evidence | 15% | Both plans captured and correctly explained |
| Replica correctness | 25% | Streaming confirmed, read-only proven, partitions replicate |
| Lag measurement | 10% | Real before/after numbers under load |
| Runbook quality | 15% | Reproducible by a stranger; every command's purpose stated |
| Reflection | 10% | Thoughtful on lag tolerance + production gaps |

---

## Why this matters

Partitioning and replication are the two scaling tools you'll actually reach for on the job, in that order, long before you ever shard. Doing both once, end to end, and *documenting them as a runbook* is exactly the artifact a senior engineer produces before a real migration. It also feeds directly into Week 12's capstone, where you tune a full database to a latency target — and a partitioned table with a read replica is a common part of hitting one.

---

## Cleanup

Stop both clusters and remove the scratch data directories when done (see Exercise 2's cleanup). Keep your `.md` and `.sql` artifacts.

When done: push, then start [Week 12 — Capstone: Tune & Design](../../week-12-capstone-tune-and-design/).
