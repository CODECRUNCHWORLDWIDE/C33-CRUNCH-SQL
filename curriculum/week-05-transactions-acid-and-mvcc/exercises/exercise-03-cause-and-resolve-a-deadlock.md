# Exercise 3 — Cause and Resolve a Deadlock

**Goal:** Deliberately deadlock two transactions, read PostgreSQL's deadlock error and understand who got killed and why, then fix the code so the deadlock can't form — using the one discipline that prevents most deadlocks: **consistent lock ordering**.

**Estimated time:** 45 minutes. **Sessions:** 2 (`psql` A and B).

## Setup (run once, either session)

```sql
DROP TABLE IF EXISTS accounts;
CREATE TABLE accounts (id int PRIMARY KEY, owner text, balance numeric);
INSERT INTO accounts VALUES (1, 'Alice', 500), (2, 'Bob', 300);
```

## Part 1 — Cause the deadlock

Two transfers running at once: A moves money **from 1 to 2**, B moves money **from 2 to 1**. Each locks its *source* row first. Run the steps in the numbered order, alternating sessions.

| Step | Session A (1 → 2) | Session B (2 → 1) |
|-----:|-------------------|-------------------|
| 1 | `BEGIN;` | `BEGIN;` |
| 2 | `UPDATE accounts SET balance=balance-10 WHERE id=1;` | |
| 3 | | `UPDATE accounts SET balance=balance-10 WHERE id=2;` |
| 4 | `UPDATE accounts SET balance=balance+10 WHERE id=2;` *(hangs — B holds row 2)* | |
| 5 | | `UPDATE accounts SET balance=balance+10 WHERE id=1;` *(hangs — A holds row 1)* |

At step 5 both sessions are waiting on each other. Within `deadlock_timeout` (default **1 second**), PostgreSQL detects the cycle and **aborts one of them** with:

```
ERROR:  deadlock detected
DETAIL:  Process NNN waits for ShareLock on transaction MMM; blocked by process PPP.
```

**Record in `solutions.md`:** which session was chosen as the victim, the full error text, and the SQLSTATE (`40P01`). Then `ROLLBACK;` (or `COMMIT;`) in the surviving session and `ROLLBACK;` in the victim to clean up.

## Part 2 — Explain it

Answer these in `solutions.md`:

1. Draw (in text is fine) the "waits-for" graph at step 5: who holds what, who wants what. Where is the cycle?
2. How long did Postgres wait before breaking it, and what setting controls that? (`SHOW deadlock_timeout;`)
3. Did the victim's earlier `UPDATE` survive? Why not?

## Part 3 — Fix it with lock ordering

The deadlock formed because A took row 1 then row 2, while B took row 2 then row 1 — **opposite orders**. The fix: make *every* transaction lock the rows it needs in the **same canonical order** (say, ascending `id`). Then a cross-cycle is impossible.

Reset balances (`UPDATE accounts SET balance=500 WHERE id=1; UPDATE accounts SET balance=300 WHERE id=2;`), then rerun the two transfers, but this time **both** sessions touch id 1 before id 2:

| Step | Session A (1 → 2) | Session B (2 → 1) |
|-----:|-------------------|-------------------|
| 1 | `BEGIN;` | `BEGIN;` |
| 2 | `UPDATE accounts SET balance=balance-10 WHERE id=1;` | |
| 3 | | `UPDATE accounts SET balance=balance+10 WHERE id=1;` *(waits for A's row-1 lock — but only waits, no cycle)* |
| 4 | `UPDATE accounts SET balance=balance+10 WHERE id=2;` | |
| 5 | `COMMIT;` *(releases row 1; B unblocks)* | |
| 6 | | `UPDATE accounts SET balance=balance-10 WHERE id=2;` `COMMIT;` |

**Expected:** no deadlock. B simply *waits* at step 3 for A to finish, then proceeds. One transaction is briefly blocked, but neither is killed. Confirm final balances are consistent (total still 800). In `solutions.md`, state the rule you just applied in one sentence.

## Part 4 — Add a defensive retry (stretch)

Lock ordering prevents the *predictable* deadlocks, but under real load some conflicts (and serialization failures) still happen, so production code retries. Sketch — in `solutions.md`, as pseudocode in any language — a loop that:

1. `BEGIN`s the transaction,
2. runs the transfer,
3. on catching SQLSTATE `40P01` (deadlock) **or** `40001` (serialization failure), rolls back and retries up to N times,
4. gives up after N with an error.

You don't have to run it — just write the shape. This is the same retry loop from Lecture 3 §7.

## Done when…

- [ ] You caused a real deadlock and captured the error text + SQLSTATE `40P01` in `solutions.md`.
- [ ] You can name the setting that controls the detection delay and its default (1s).
- [ ] Your Part 3 rerun completes with **no** deadlock and correct final balances.
- [ ] You can state the lock-ordering rule in one sentence.
- [ ] You wrote a retry-loop sketch that handles both `40P01` and `40001`.

## Submission

Commit `solutions.md` (all four parts) to `c33-week-05/exercise-03/`.
