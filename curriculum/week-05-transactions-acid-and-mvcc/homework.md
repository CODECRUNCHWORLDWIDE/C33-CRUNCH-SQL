# Week 5 — Homework

Six practice tasks (~4 hours total) that reinforce the lectures and stretch you past the exercises. Do them after the lectures and exercises, before the quiz. Keep answers and pasted SQL output in a `homework.md` in your portfolio under `c33-week-05/`.

Assume PostgreSQL 16 unless a task says otherwise. Reset the seed table between tasks as needed:

```sql
DROP TABLE IF EXISTS accounts;
CREATE TABLE accounts (id int PRIMARY KEY, owner text, balance numeric NOT NULL CHECK (balance >= 0));
INSERT INTO accounts VALUES (1, 'Alice', 500), (2, 'Bob', 300), (3, 'Carol', 900);
```

## 1. ACID in your own words (~20 min)

Write one paragraph per letter of ACID, each anchored to the `accounts` table:

- **A** — give a two-statement transaction where atomicity matters and say what corrupts without it.
- **C** — name the constraint on `accounts` that consistency enforces, and a transaction it would reject.
- **I** — describe what a second session sees while your transfer is mid-flight (uncommitted).
- **D** — explain what "durability" promises the instant `COMMIT` returns.

Don't recite definitions — tie each to the table.

## 2. Savepoint surgery (~30 min)

Write a **single transaction** that:

1. inserts Dave (id 4, balance 100),
2. sets a savepoint,
3. attempts to overdraw Dave by 500 (which violates the `CHECK`),
4. recovers from the failure via `ROLLBACK TO SAVEPOINT` **without aborting the whole transaction**,
5. instead deducts a valid 50 from Dave,
6. commits.

Paste the transaction and the final `SELECT * FROM accounts WHERE id=4`. Then answer: why couldn't you have done step 4's recovery with a plain `ROLLBACK`?

## 3. Fill in the anomaly/level matrix from memory (~15 min)

Reproduce this table and fill every cell with "possible" or "prevented" **for PostgreSQL's actual behavior** (not the ANSI standard). Then, below it, note the two places Postgres is *stricter* than the standard.

| Level | Dirty read | Non-repeatable read | Phantom | Write skew |
|-------|:----------:|:-------------------:|:-------:|:----------:|
| READ COMMITTED | | | | |
| REPEATABLE READ | | | | |
| SERIALIZABLE | | | | |

## 4. Read a live snapshot (~30 min)

In one session:

```sql
SELECT pg_current_xact_id();      -- your transaction id
SELECT xmin, xmax, ctid, * FROM accounts WHERE id = 1;
UPDATE accounts SET balance = balance + 1 WHERE id = 1;
SELECT xmin, xmax, ctid, * FROM accounts WHERE id = 1;
```

Paste both `xmin/xmax/ctid` rows. In prose, answer:

- Why did `ctid` change after the `UPDATE`?
- What happened to the *old* tuple (is it gone)? What eventually reclaims it?
- Using `xmin` and the visibility rule, explain in one sentence why an uncommitted `UPDATE` from another session would be invisible to you.

## 5. Design against a lost update (~45 min)

The service increments a shared `page_views` counter under heavy concurrency. Write **two** correct implementations and paste the SQL:

1. an **atomic in-place** version (`UPDATE … SET value = value + 1 …`), and
2. an **optimistic** version using a `version` column and a conditional `UPDATE … WHERE version = :v` with a described retry.

Then answer: for a counter hit millions of times a day, which do you ship and why? Which would you pick instead if the new value came from a slow computation done in application code, and why?

## 6. Deadlock post-mortem (~40 min)

Reproduce the two-session deadlock from Exercise 3 (A locks row 1 then wants row 2; B locks row 2 then wants row 1). Then write a short post-mortem answering:

- Which session was the victim, and what SQLSTATE did it get?
- Draw the waits-for cycle in text.
- What single coding discipline would have prevented it, and rewrite one of the two transactions to follow it.
- Even with that discipline, why should the application *still* wrap the transaction in a retry loop? What two error codes should that loop catch?

## Submission

Commit `homework.md` (with pasted SQL and output for every task) to `c33-week-05/`. Aim to finish before attempting the [quiz](./quiz.md).
