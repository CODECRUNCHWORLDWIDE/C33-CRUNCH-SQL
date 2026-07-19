# Exercise 2 — Reproduce an Anomaly at Each Isolation Level

**Goal:** With two live sessions, reproduce a **non-repeatable read** and a **phantom** at `READ COMMITTED`, watch both vanish at `REPEATABLE READ`, produce **write skew** at `REPEATABLE READ`, and watch `SERIALIZABLE` catch it. Along the way, prove to yourself that PostgreSQL will *not* give you a dirty read.

**Estimated time:** 60 minutes. **Sessions:** 2 (`psql` A and B, side by side).

## Setup (run once, either session)

```sql
DROP TABLE IF EXISTS accounts, on_call;
CREATE TABLE accounts (id int PRIMARY KEY, owner text, balance numeric);
INSERT INTO accounts VALUES (1, 'Alice', 500), (2, 'Bob', 300);

CREATE TABLE on_call (doctor text PRIMARY KEY, on_call boolean);
INSERT INTO on_call VALUES ('Alice', true), ('Bob', true);
```

Throughout: **A** = first prompt, **B** = second prompt. Run the numbered steps *in order across both columns* — the interleaving is the whole point. Record each result in `solutions.md`.

## Part 1 — Non-repeatable read (READ COMMITTED)

| Step | Session A | Session B |
|-----:|-----------|-----------|
| 1 | `BEGIN;` *(default is READ COMMITTED)* | |
| 2 | `SELECT balance FROM accounts WHERE id=1;` | |
| 3 | | `UPDATE accounts SET balance=999 WHERE id=1;` |
| 4 | | `COMMIT;` |
| 5 | `SELECT balance FROM accounts WHERE id=1;` | |
| 6 | `COMMIT;` | |

**Expected:** step 2 returns **500**, step 5 returns **999** — the same query, two answers, inside one transaction. Record both. That's a non-repeatable read.

## Part 2 — The same steps at REPEATABLE READ

Reset: `UPDATE accounts SET balance=500 WHERE id=1;` (either session, autocommit).

Repeat Part 1 exactly, but change A's step 1 to:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
```

**Expected:** step 5 now returns **500**, not 999 — A holds its opening snapshot for the whole transaction. In `solutions.md`, state in one sentence *why* the answer stopped changing.

## Part 3 — Phantom read (READ COMMITTED), then blocked

| Step | Session A (`READ COMMITTED`) | Session B |
|-----:|------------------------------|-----------|
| 1 | `BEGIN;` | |
| 2 | `SELECT count(*) FROM accounts WHERE balance > 200;` | |
| 3 | | `INSERT INTO accounts VALUES (3,'Carol',700); COMMIT;` |
| 4 | `SELECT count(*) FROM accounts WHERE balance > 200;` | |
| 5 | `COMMIT;` | |

**Expected:** step 2 = **2**, step 4 = **3** — a row appeared in your result set: a phantom. Now delete Carol (`DELETE FROM accounts WHERE id=3;`) and rerun with A at `REPEATABLE READ`. Step 4 should stay **2**. Record both runs and note: the standard *allows* phantoms at REPEATABLE READ, but Postgres forbids them — what does that tell you about Postgres's snapshots?

## Part 4 — Prove Postgres won't dirty-read

| Step | Session A | Session B |
|-----:|-----------|-----------|
| 1 | | `BEGIN;` |
| 2 | | `UPDATE accounts SET balance=1 WHERE id=1;` *(no commit!)* |
| 3 | `SELECT balance FROM accounts WHERE id=1;` | |
| 4 | | `ROLLBACK;` |

**Expected:** step 3 returns the last **committed** value (500), **never** the uncommitted `1`. Try to force a dirty read by starting A with `BEGIN ISOLATION LEVEL READ UNCOMMITTED;` and repeating — you'll get the identical result. In `solutions.md`, answer: **why is a dirty read impossible in Postgres, in terms of snapshots and tuple visibility (`xmin` must have committed)?**

## Part 5 — Write skew (REPEATABLE READ) and SERIALIZABLE catching it

The on-call invariant: *at least one doctor must stay on call.* Reset first: `UPDATE on_call SET on_call=true;`.

| Step | Session A (`REPEATABLE READ`) | Session B (`REPEATABLE READ`) |
|-----:|-------------------------------|-------------------------------|
| 1 | `BEGIN ISOLATION LEVEL REPEATABLE READ;` | `BEGIN ISOLATION LEVEL REPEATABLE READ;` |
| 2 | `SELECT count(*) FROM on_call WHERE on_call;` | `SELECT count(*) FROM on_call WHERE on_call;` |
| 3 | `UPDATE on_call SET on_call=false WHERE doctor='Alice';` | `UPDATE on_call SET on_call=false WHERE doctor='Bob';` |
| 4 | `COMMIT;` | `COMMIT;` |

**Expected:** both step 2s return **2**, both commits **succeed**, and now `SELECT count(*) FROM on_call WHERE on_call;` returns **0** — the invariant is broken though each transaction was individually correct. That's write skew.

Now reset (`UPDATE on_call SET on_call=true;`) and run the exact same table with **`SERIALIZABLE`** on *both* sessions (change step 1 to `BEGIN ISOLATION LEVEL SERIALIZABLE;`). **Expected:** the second `COMMIT` fails with:

```
ERROR:  could not serialize access due to read/write dependencies among transactions
```

Record the error and its SQLSTATE (`40001`). In `solutions.md`, answer: **whose responsibility is it to recover from a `40001`, and what should that code do?**

## Done when…

- [ ] `solutions.md` shows the two different answers in Part 1 and the stable answer in Part 2.
- [ ] You reproduced a phantom in Part 3 and showed it disappear at REPEATABLE READ.
- [ ] You demonstrated that Part 4 never sees the uncommitted value, at any level, and explained why in snapshot terms.
- [ ] You produced write skew (final count = 0) at REPEATABLE READ and captured the `40001` error at SERIALIZABLE.
- [ ] You can fill in this sentence: "READ COMMITTED allows ___ and ___; REPEATABLE READ still allows ___; only SERIALIZABLE forbids ___."

## Submission

Commit `solutions.md` with all five parts to `c33-week-05/exercise-02/`.
