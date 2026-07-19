# Challenge 1 — Scale a write-heavy workload

**Time:** ~90 minutes. **Difficulty:** Hard. **Deliverable:** a design doc, `scaling-plan.md`.

## The scenario

You run the backend for **PulseMetrics**, an IoT telemetry platform. Devices in the field POST readings to your API, which writes them to one PostgreSQL 16 primary. Today:

- **Ingest rate:** 40,000 inserts/second, growing 15%/month. Each row: `device_id`, `ts`, `metric`, `value` (~60 bytes).
- **Table:** `readings`, now 8 TB, single unpartitioned heap. `INSERT`s are slowing; autovacuum can't keep up; the disk is 78% full.
- **Reads:** dashboards query "last 24h for device X" (frequent) and "hourly rollups for device X over 90 days" (frequent). Occasionally: "fleet-wide average for metric Y, last hour" (rare).
- **Retention:** raw readings older than 90 days are deleted nightly — a `DELETE` that currently runs for **6 hours** and bloats the table.
- **Availability:** the product has a 2 AM maintenance window today; the team wants to move to near-zero downtime.

The primary is a big box already (64 cores, 512 GB RAM, fast NVMe). Vertical scaling has maybe one more doubling in it, at a steep price.

## Your job

Write `scaling-plan.md` proposing a path to handle 4× the current write rate over the next 18 months, in the **right order of operations**. Cover:

1. **The immediate bleeding.** The nightly 6-hour `DELETE` is the most acute pain and the easiest win. What's your fix, and roughly how much does it cut the retention job? (Think about Lecture 1.)
2. **Partitioning plan.** How do you partition `readings`? State the strategy (range/list/hash), the key, the granularity (daily? monthly?), and how partitions get created and expired automatically. Show the `CREATE TABLE` for the parent and one partition.
3. **Read offloading.** Which reads move to a replica, and which must stay on the primary? For the dashboard queries, is replication lag acceptable? Justify per-query.
4. **The write ceiling.** Partitioning + a bigger box buys time, but at 4× you may exhaust one primary's write capacity. At what signal do you shard, and what's your shard key? Defend it against the obvious alternative. (The device fleet is uneven — a few thousand chatty devices produce most of the volume. How does that shape the key?)
5. **The rollup path.** The "hourly rollups over 90 days" query shouldn't scan raw readings every time. Propose a continuous aggregation / rollup table strategy so dashboards read pre-aggregated data.
6. **Migration without the maintenance window.** How do you move an 8 TB live table to a partitioned scheme without taking the product down? (Hint: new writes to a new partitioned table, backfill old data in batches, cut over.)

## Constraints

- Free/open tools only (PostgreSQL 16, its extensions, PgBouncer). You may *mention* Citus or TimescaleDB and Patroni, but explain what each buys you — don't just name-drop.
- You must **order** the work: what ships first, second, third, and why. A plan that shards on day one loses.
- Call out at least two things that could go wrong and how you'd detect them.

## Hints

<details>
<summary>Where to start</summary>

The retention `DELETE` → `DROP PARTITION` swap is the single highest-leverage change and it *requires* partitioning, so partitioning is step one. Range-partition by time (daily, given the volume). Only after partitioning + replicas are exhausted do you consider sharding by `device_id` (hash), keeping a device's history together. TimescaleDB (a Postgres extension) automates the partitioning + rollups if you want to argue for it — but say what it costs (another dependency).

</details>

<details>
<summary>The uneven-fleet trap</summary>

If a few thousand devices produce most of the volume, hashing `device_id` still spreads *devices* evenly but a single chatty device stays whole on one shard. That's usually fine (no single device out-writes a shard), unlike the workspace-whale problem where one tenant *can* dwarf a shard. Explain why the shapes differ.

</details>

## How success is judged

| Criterion | What "great" looks like |
|-----------|-------------------------|
| Order of operations | Cheap wins first (partition + drop-partition retention), sharding genuinely last with a stated trigger |
| Partitioning design | Correct key, sane granularity, automated creation + expiry |
| Read/write split | Per-query reasoning about lag tolerance, not a blanket rule |
| Migration realism | A believable zero-downtime cutover, not "take a maintenance window" |
| Risk awareness | Names concrete failure modes and how to detect them |

## Submission

Commit `scaling-plan.md` to `c33-week-11/challenge-01/`. Two pages max — senior engineers write tight docs.
