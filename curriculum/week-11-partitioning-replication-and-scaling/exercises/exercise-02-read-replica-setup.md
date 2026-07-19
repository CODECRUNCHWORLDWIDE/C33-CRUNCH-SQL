# Exercise 2 — Read-replica setup

**Goal:** Stand up a real physical streaming replica of a PostgreSQL 16 primary, watch WAL stream to it, confirm it's read-only, and measure replication lag. You'll run **two clusters on one machine** (different data directories and ports) so you can do this on a laptop.

**Estimated time:** 75 minutes. **Stack:** PostgreSQL 16. **You need:** command-line access to `initdb`, `pg_ctl`, `pg_basebackup`, and permission to start two clusters.

> This is the load-bearing exercise for the mini-project. If you get stuck, read Lecture 2 §4 alongside it — the steps map one-to-one.

## Mental model first

A replica is a server continuously replaying the primary's WAL. To build one you: (1) let the primary accept replication connections, (2) clone the primary's data directory with `pg_basebackup`, (3) start the clone in standby mode, (4) verify it's streaming and read-only.

## Step 1 — Create and start the PRIMARY

```bash
export PRIMARY=/tmp/c33/primary
initdb -D "$PRIMARY"
```

Edit `$PRIMARY/postgresql.conf`:

```ini
port = 5433
wal_level = replica
max_wal_senders = 10
listen_addresses = 'localhost'
```

Allow a replication connection in `$PRIMARY/pg_hba.conf` (local dev, trust is fine here):

```conf
local   replication   all                 trust
host    replication   all   127.0.0.1/32  trust
```

Start it and create a replication role + a slot + some data:

```bash
pg_ctl -D "$PRIMARY" -l "$PRIMARY/log" start

psql -p 5433 -d postgres -c "CREATE ROLE replicator WITH REPLICATION LOGIN;"
psql -p 5433 -d postgres -c "SELECT pg_create_physical_replication_slot('replica1_slot');"
psql -p 5433 -d postgres -c "CREATE TABLE t (id int PRIMARY KEY, note text);"
psql -p 5433 -d postgres -c "INSERT INTO t VALUES (1,'from primary before backup');"
```

## Step 2 — Clone the primary into a REPLICA

`pg_basebackup` copies the primary's data directory over a replication connection. `-R` writes the standby config for you.

```bash
export REPLICA=/tmp/c33/replica
pg_basebackup -h 127.0.0.1 -p 5433 -U replicator \
    -D "$REPLICA" -R -X stream -S replica1_slot
```

Confirm `-R` did its job:

```bash
ls "$REPLICA/standby.signal"                    # exists → boots as standby
grep primary_conninfo "$REPLICA/postgresql.auto.conf"
```

Give the replica its own port:

```ini
# append to $REPLICA/postgresql.conf
port = 5434
hot_standby = on
```

## Step 3 — Start the replica and verify streaming

```bash
pg_ctl -D "$REPLICA" -l "$REPLICA/log" start
```

On the **primary**, confirm the replica connected and is streaming:

```bash
psql -p 5433 -d postgres -c \
  "SELECT client_addr, state, sync_state,
          pg_wal_lsn_diff(sent_lsn, replay_lsn) AS bytes_behind
   FROM pg_stat_replication;"
```

`state = streaming` means it worked. `sync_state = async` is expected (we didn't configure synchronous replication).

## Step 4 — Watch data flow

Write on the primary, read on the replica:

```bash
psql -p 5433 -d postgres -c "INSERT INTO t VALUES (2,'written after replica started');"
sleep 1
psql -p 5434 -d postgres -c "SELECT * FROM t ORDER BY id;"      # replica sees BOTH rows
```

## Step 5 — Prove the replica is read-only

```bash
psql -p 5434 -d postgres -c "INSERT INTO t VALUES (3,'should fail');"
# ERROR:  cannot execute INSERT in a read-only transaction
```

That error is the point of a read replica — it serves reads, never writes.

## Step 6 — Measure replication lag

Ask the replica how far behind it is right now:

```bash
psql -p 5434 -d postgres -c \
  "SELECT now() - pg_last_xact_replay_timestamp() AS replica_lag;"
```

On an idle laptop this is tiny. To *see* lag, hammer the primary in one terminal and re-run the lag query in another:

```bash
psql -p 5433 -d postgres -c \
  "INSERT INTO t SELECT g, 'bulk' FROM generate_series(100,500000) g;"
```

Watch `replica_lag` rise, then fall back to near-zero as the replica catches up. That gap is exactly the window in which a read replica would serve **stale** data — the eventual-consistency reality from Lecture 2 §6.

## Expected result

- Two clusters running (ports 5433 primary, 5434 replica).
- `pg_stat_replication` shows one streaming standby.
- The replica mirrors the primary's data and rejects writes.
- You observed lag rise and fall under a bulk write.

## Done when…

- [ ] `pg_stat_replication` on the primary shows `state = streaming`.
- [ ] The replica returns rows written on the primary *after* the replica started.
- [ ] An `INSERT` on the replica fails with a read-only error.
- [ ] You watched `replica_lag` grow under load and shrink afterward.
- [ ] You can explain, in one sentence, what a replication **slot** prevented in this setup.

## Stretch

- Make the replica **synchronous**: on the primary set `synchronous_standby_names = 'walreceiver'` (or the standby's `application_name`) and `synchronous_commit = on`, reload, and note how commit latency changes. Then stop the replica and watch primary writes **block** — the availability cost of synchronous replication, live.
- Promote the replica: `pg_ctl -D "$REPLICA" promote`, then insert into it. It's now a primary. Discuss why doing this automatically (real failover) needs quorum to avoid split-brain.

## Cleanup

```bash
pg_ctl -D "$REPLICA" stop
pg_ctl -D "$PRIMARY" stop
rm -rf /tmp/c33
```

Commit your `pg_stat_replication` output and lag observations to `c33-week-11/exercise-02/`.
