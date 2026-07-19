# Exercise 1 — Transactions and Savepoints

**Goal:** Drive a transaction by hand, feel what `ROLLBACK` undoes, watch an error poison a transaction, and use `SAVEPOINT` to recover part of a transaction without losing the rest.

**Estimated time:** 40 minutes. **Sessions:** 1 (single `psql`).

## Setup

```sql
DROP TABLE IF EXISTS accounts;
CREATE TABLE accounts (
    id      int PRIMARY KEY,
    owner   text NOT NULL,
    balance numeric NOT NULL CHECK (balance >= 0)
);
INSERT INTO accounts VALUES (1, 'Alice', 500), (2, 'Bob', 300);
```

## Part A — commit vs. rollback

1. Start a transaction and move $100 from Alice to Bob:

   ```sql
   BEGIN;
   UPDATE accounts SET balance = balance - 100 WHERE id = 1;
   UPDATE accounts SET balance = balance + 100 WHERE id = 2;
   SELECT id, owner, balance FROM accounts ORDER BY id;
   ```

   Note the balances your session sees. **Do not commit yet.**

2. Now change your mind:

   ```sql
   ROLLBACK;
   SELECT id, owner, balance FROM accounts ORDER BY id;
   ```

   Confirm the balances are back to 500 / 300. Write one sentence in `solutions.md`: what did `ROLLBACK` undo, and how many of the two `UPDATE`s did it reverse?

3. Do the transfer again, but this time `COMMIT;`. Confirm the change is now permanent (re-run the `SELECT`). This is the difference the whole week rests on: **nothing is real until `COMMIT`.**

## Part B — an error poisons the transaction

4. Start a fresh transaction and deliberately cause an error partway through:

   ```sql
   BEGIN;
   UPDATE accounts SET balance = balance - 50 WHERE id = 1;
   INSERT INTO accounts VALUES (1, 'Duplicate', 0);   -- violates PRIMARY KEY
   SELECT * FROM accounts;                              -- try to keep working
   ```

5. Read the two error messages. The `INSERT` fails with a duplicate-key error; the `SELECT` then fails with `current transaction is aborted…`. In `solutions.md`, answer: **once a statement errors, what state is a PostgreSQL transaction in, and what is the only way out?**

6. Get out and confirm the `UPDATE` from step 4 did **not** stick:

   ```sql
   ROLLBACK;
   SELECT id, owner, balance FROM accounts WHERE id = 1;   -- still 400 from Part A, not 350
   ```

## Part C — savepoints: recover part of a transaction

Here's the payoff. You want to run several inserts as one transaction, but if *one* fails you'd like to skip just that one and keep the others — without starting over.

7. Run this exactly, reading the comments:

   ```sql
   BEGIN;
   INSERT INTO accounts VALUES (3, 'Carol', 1000);

   SAVEPOINT before_dave;
   INSERT INTO accounts VALUES (3, 'Dave', 200);   -- FAILS: id 3 already used
   -- transaction is now aborted, but only back to the savepoint's reach
   ROLLBACK TO SAVEPOINT before_dave;               -- rewind; transaction lives on

   INSERT INTO accounts VALUES (4, 'Dave', 200);    -- fix the id; this succeeds
   COMMIT;
   ```

8. Confirm the outcome:

   ```sql
   SELECT id, owner, balance FROM accounts ORDER BY id;
   ```

   You should have Carol (id 3) **and** Dave (id 4), and **no** trace of the failed Dave-at-id-3 insert. In `solutions.md`: explain what `ROLLBACK TO SAVEPOINT before_dave` kept and what it discarded.

## Part D — nested savepoints (stretch)

9. Predict, then verify. What does the final `SELECT` show?

   ```sql
   BEGIN;
   UPDATE accounts SET balance = 1 WHERE id = 1;
   SAVEPOINT s1;
     UPDATE accounts SET balance = 2 WHERE id = 1;
     SAVEPOINT s2;
       UPDATE accounts SET balance = 3 WHERE id = 1;
     ROLLBACK TO SAVEPOINT s1;      -- discards s2's work AND the balance=2 update
   SELECT balance FROM accounts WHERE id = 1;   -- predict this before running
   COMMIT;
   ```

   Write your prediction first, then run it. If you predicted `1`, you understand savepoint nesting: rolling back to `s1` discards everything after `s1`, including the inner savepoint `s2` and its update.

## Done when…

- [ ] `solutions.md` shows the balances before and after a `ROLLBACK` and after a `COMMIT`.
- [ ] You can state, in one sentence, what state a transaction is in after any statement errors, and how to leave it.
- [ ] Your Part C result has Carol at id 3 and Dave at id 4, with no failed insert visible.
- [ ] You correctly predicted the Part D result (`balance = 1`) *before* running it.
- [ ] You can explain the difference between `ROLLBACK` (whole transaction) and `ROLLBACK TO SAVEPOINT` (back to a marker, transaction continues).

## Submission

Commit `solutions.md` to `c33-week-05/exercise-01/`.
