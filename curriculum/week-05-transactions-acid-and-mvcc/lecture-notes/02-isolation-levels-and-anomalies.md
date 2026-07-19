# Lecture 2 — Isolation Levels and the Anomalies They Allow

> **Duration:** ~2 hours. **Outcome:** You can name the four anomalies, name the four isolation levels, state which level allows which anomaly, and reproduce each one in two `psql` sessions — including the ones PostgreSQL refuses to let happen.

Isolation is the "I" in ACID, and it is the one you actually get to tune. The other three are essentially on-or-off. Isolation is a **dial** with four settings, and each setting is a bargain: a higher level forbids more anomalies but permits less concurrency. To turn the dial on purpose you need two vocabularies — the *anomalies* (the bad things that can happen) and the *levels* (how much protection you bought). This lecture pairs them.

## 1. Why weaker isolation exists at all

If perfect isolation (every transaction behaves as if it ran completely alone) is correct, why would anyone choose less? **Speed.** Perfect isolation requires the database to detect and prevent every possible interference between concurrent transactions, which means more locking or more conflict-checking, which means transactions wait on each other. On a busy system, the strictest level can cut throughput dramatically and cause transactions to abort and retry.

So the SQL standard defines weaker levels that permit specific anomalies. The deal is explicit: *"I will let this particular bad thing happen, in exchange for going faster — because in my workload that bad thing either can't occur or doesn't matter."* Making that deal *knowingly* is the entire skill. Making it by accident is where data corruption comes from.

## 2. The four anomalies

An **anomaly** is an outcome that could not happen if the transactions had run one after another. The SQL standard names four (the first three are the classic "phenomena"; the fourth is broader).

| Anomaly | What happens | Plain-English example |
|---------|--------------|-----------------------|
| **Dirty read** | You read a row another transaction wrote but hasn't committed. | You see Bob's balance as 400 mid-transfer; then that transfer rolls back and it was really 300. You acted on money that never moved. |
| **Non-repeatable read** | You read a row twice in one transaction and get *different values*, because another transaction committed an `UPDATE` in between. | You read a price as $10, do some work, read it again and it's $12 — inside a single transaction. |
| **Phantom read** | You run the *same* query twice and the *set of rows* changes, because another transaction committed an `INSERT`/`DELETE` matching your `WHERE`. | `SELECT count(*) WHERE status='open'` returns 5, then 6 — a new matching row appeared. |
| **Serialization anomaly** (incl. **write skew**) | Each transaction individually respects the rules, but their *combination* leaves the database in a state no serial order could produce. | Two doctors each go off-call, each checking "someone else is still on call." Both checks pass; now nobody is on call. |

Notice the escalation: dirty read is about *uncommitted* data; non-repeatable and phantom are about *committed* data changing under you; serialization anomaly is about two individually-correct transactions producing a jointly-wrong result. Each level below buys protection from a longer prefix of this list.

### Lost update — the fifth one you'll actually cause

The standard folds **lost update** into the others, but it deserves its own name because it is the anomaly you will personally write into production. Two transactions read the same value, each computes a new value from what it read, and each writes back. One write silently overwrites the other:

```
T1: read balance = 500
T2: read balance = 500
T1: write 500 - 100 = 400
T2: write 500 - 200 = 300   ← T1's deduction is gone; $100 vanished
```

We spend all of [Challenge 2](../challenges/challenge-02-fix-a-lost-update.md) killing this bug. Keep it in mind through the whole lecture.

## 3. The four isolation levels

The SQL standard defines four levels by which anomalies they *forbid*. Here is the standard's table — memorize the shape of it:

| Isolation level | Dirty read | Non-repeatable read | Phantom | Serialization anomaly |
|-----------------|:----------:|:-------------------:|:-------:|:---------------------:|
| **READ UNCOMMITTED** | possible | possible | possible | possible |
| **READ COMMITTED** | — | possible | possible | possible |
| **REPEATABLE READ** | — | — | possible | possible |
| **SERIALIZABLE** | — | — | — | — |

"possible" = the standard *allows* it at that level; "—" = forbidden. Read it as a staircase: each level down forbids one more anomaly.

### What PostgreSQL actually implements

Here is the crucial twist that trips up people who memorized the standard: **PostgreSQL's real behavior is stricter than the standard requires.** Postgres has only *three* distinct levels internally, because its MVCC design makes some anomalies impossible to produce even at weaker settings.

| You ask for | Postgres gives you | Dirty read | Non-repeatable | Phantom | Serialization anomaly |
|-------------|--------------------|:----------:|:--------------:|:-------:|:---------------------:|
| `READ UNCOMMITTED` | behaves as READ COMMITTED | **never** | possible | possible | possible |
| `READ COMMITTED` *(default)* | READ COMMITTED | never | possible | possible | possible |
| `REPEATABLE READ` | snapshot isolation | never | never | **never** | possible |
| `SERIALIZABLE` | serializable snapshot isolation (SSI) | never | never | never | never |

Two things to burn in:

1. **Postgres never gives dirty reads.** Ask for `READ UNCOMMITTED` and you still get `READ COMMITTED`. There is no way to see uncommitted data in Postgres. (So in Exercise 2, "reproduce a dirty read" becomes "demonstrate that you *can't* — and explain why.")
2. **Postgres `REPEATABLE READ` also forbids phantoms.** The standard permits phantoms at REPEATABLE READ; Postgres's snapshot isolation forbids them too. So the only anomaly left at Postgres REPEATABLE READ is the serialization anomaly (write skew).

Set the level per transaction:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
-- … work …
COMMIT;

-- or set it after BEGIN, before any query touches data:
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

Check where you are with `SHOW transaction_isolation;`. The database-wide default is `read committed` and you rarely change that globally — you raise the level on the specific transactions that need it.

## 4. Reproductions — run these in two sessions

Open **two** `psql` sessions on the same database. We'll call them **A** and **B**. Seed:

```sql
-- run once, in either session
DROP TABLE IF EXISTS accounts;
CREATE TABLE accounts (id int PRIMARY KEY, owner text, balance numeric);
INSERT INTO accounts VALUES (1, 'Alice', 500), (2, 'Bob', 300);
```

### 4a. Non-repeatable read (allowed at READ COMMITTED)

| Step | Session A (`READ COMMITTED`) | Session B |
|-----:|------------------------------|-----------|
| 1 | `BEGIN;` | |
| 2 | `SELECT balance FROM accounts WHERE id=1;` → **500** | |
| 3 | | `UPDATE accounts SET balance=999 WHERE id=1;` `COMMIT;` |
| 4 | `SELECT balance FROM accounts WHERE id=1;` → **999** | |
| 5 | `COMMIT;` | |

A read the same row twice in one transaction and got two different answers. That's a non-repeatable read. It happens at READ COMMITTED because each statement in A takes a *fresh snapshot* — it sees whatever was committed the moment that statement started.

### 4b. Prevent it — REPEATABLE READ

Redo the exact steps but start A with `BEGIN ISOLATION LEVEL REPEATABLE READ;`. Now step 4 still returns **500**. Under REPEATABLE READ, A took one snapshot at its first query and holds it for the whole transaction; B's committed change is invisible to A until A commits and starts fresh. The non-repeatable read is gone.

### 4c. Phantom (allowed at READ COMMITTED, gone at REPEATABLE READ)

| Step | Session A (`READ COMMITTED`) | Session B |
|-----:|------------------------------|-----------|
| 1 | `BEGIN;` | |
| 2 | `SELECT count(*) FROM accounts WHERE balance > 200;` → **2** | |
| 3 | | `INSERT INTO accounts VALUES (3,'Carol',700);` `COMMIT;` |
| 4 | `SELECT count(*) FROM accounts WHERE balance > 200;` → **3** | |
| 5 | `COMMIT;` | |

The row set grew under A — a phantom. Rerun with A at REPEATABLE READ and step 4 stays **2**: Postgres's snapshot hides Carol from A entirely.

### 4d. Dirty read — prove Postgres won't

| Step | Session A | Session B |
|-----:|-----------|-----------|
| 1 | | `BEGIN;` `UPDATE accounts SET balance=1 WHERE id=1;` *(no commit)* |
| 2 | `SELECT balance FROM accounts WHERE id=1;` → **500** (the old committed value) | |
| 3 | | `ROLLBACK;` |

A never saw the uncommitted `1`. Set A to `READ UNCOMMITTED` and repeat — identical result. **You cannot make Postgres dirty-read.** This is a feature: MVCC (Lecture 3) hands every reader a consistent committed snapshot, so uncommitted rows are simply never visible.

### 4e. Serialization anomaly — write skew (allowed at REPEATABLE READ, blocked at SERIALIZABLE)

The classic: an on-call roster where the rule is "at least one doctor must be on call." Two doctors each try to go off-call at the same time.

```sql
DROP TABLE IF EXISTS on_call;
CREATE TABLE on_call (doctor text PRIMARY KEY, on_call boolean);
INSERT INTO on_call VALUES ('Alice', true), ('Bob', true);
```

| Step | Session A (`REPEATABLE READ`) | Session B (`REPEATABLE READ`) |
|-----:|-------------------------------|-------------------------------|
| 1 | `BEGIN ISOLATION LEVEL REPEATABLE READ;` | `BEGIN ISOLATION LEVEL REPEATABLE READ;` |
| 2 | `SELECT count(*) FROM on_call WHERE on_call;` → **2** | `SELECT count(*) FROM on_call WHERE on_call;` → **2** |
| 3 | `UPDATE on_call SET on_call=false WHERE doctor='Alice';` | `UPDATE on_call SET on_call=false WHERE doctor='Bob';` |
| 4 | `COMMIT;` | `COMMIT;` |

Both saw "2 on call, safe to leave," both left, **both commits succeed** — and now zero doctors are on call. No serial order (A-then-B or B-then-A) could produce that: whoever went second would have seen a count of 1 and been blocked by their own check. Their combination broke the invariant. That's a serialization anomaly (specifically write skew — they wrote *different* rows, so no lock conflicts, so REPEATABLE READ's snapshot check never fires).

Now raise both to **SERIALIZABLE** and rerun. One transaction commits; the other fails at `COMMIT` with:

```
ERROR:  could not serialize access due to read/write dependencies among transactions
```

That is SQLSTATE `40001`. Postgres's Serializable Snapshot Isolation detected that the two transactions formed a dangerous read-write cycle and aborted one to preserve serializability. **The application's job is to catch `40001` and retry the whole transaction** — we build that retry loop in Lecture 3 and Exercise 3.

## 5. So which level should I use?

A decision guide you'll refine in [Challenge 1](../challenges/challenge-01-pick-the-isolation-level.md):

| Situation | Level | Why |
|-----------|-------|-----|
| Ordinary OLTP: single-row reads/writes, atomic `UPDATE … SET x = x - n` | **READ COMMITTED** (default) | Cheapest; the anomalies it allows rarely matter for single-row atomic writes. |
| A report or batch that must see one consistent snapshot across many queries | **REPEATABLE READ** | One snapshot for the whole transaction; no numbers shifting mid-report. |
| Multi-row invariants — "at least one on call", "total must stay ≤ limit", inventory across rows | **SERIALIZABLE** | Only level that catches write skew. Pair with a retry loop. |

The default is READ COMMITTED for a reason: most transactions are single-statement or use atomic in-place arithmetic, where its anomalies can't bite. Raise the level *for the specific transactions* whose correctness depends on a multi-statement or multi-row invariant — not globally, or you'll pay the contention everywhere.

## 6. SQLite, briefly

SQLite doesn't expose isolation levels the way Postgres does — with one writer at a time, it is effectively **serializable** for writers already. Its knob is *concurrency of readers*: default rollback-journal mode blocks readers during a write; **WAL mode** (`PRAGMA journal_mode=WAL;`) lets many readers proceed against the last committed snapshot while one writer works. So SQLite trades Postgres's rich level menu for a simpler "one writer, serializable" model — fewer footguns, less write throughput.

## 7. Check yourself

- Define dirty read, non-repeatable read, phantom, and write skew in one sentence each.
- Which anomaly can you *never* reproduce in PostgreSQL, at any level, and why?
- The standard allows phantoms at REPEATABLE READ. Does Postgres? What does that tell you about Postgres's REPEATABLE READ?
- At which level does write skew first get caught? What error code and SQLSTATE do you get?
- Whose job is it to handle a `40001` serialization failure — the database's or the application's?
- Why is READ COMMITTED a sane *default* even though it allows three of the four anomalies?

When those land, run [Exercise 2 — reproduce an anomaly](../exercises/exercise-02-reproduce-an-anomaly.md).

## Further reading

- **PostgreSQL — Transaction Isolation:** <https://www.postgresql.org/docs/16/transaction-iso.html>
- **PostgreSQL — `SET TRANSACTION`:** <https://www.postgresql.org/docs/16/sql-set-transaction.html>
- **Berenson et al., "A Critique of ANSI SQL Isolation Levels" (1995)** — the paper that showed the standard's definitions are ambiguous: <https://www.microsoft.com/en-us/research/publication/a-critique-of-ansi-sql-isolation-levels/>
- **Martin Kleppmann, *Designing Data-Intensive Applications*, Ch. 7** — the clearest modern treatment of weak isolation and write skew.
