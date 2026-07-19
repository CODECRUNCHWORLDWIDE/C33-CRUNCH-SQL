# Week 5 — Quiz

Fourteen questions. Lectures closed. Aim for 12/14 before Week 6. Mix of multiple-choice and short-answer; the answer key is at the bottom in a collapsed block — attempt every question before revealing it.

---

**Q1.** ACID's "A" stands for atomicity. Which statement best describes it?

- A) Committed data survives a crash.
- B) All statements in a transaction succeed, or none do.
- C) Concurrent transactions don't see each other's uncommitted work.
- D) Constraints hold at commit time.

---

**Q2.** You run a bare `UPDATE accounts SET balance = 0 WHERE id = 1;` with no `BEGIN`. Is it in a transaction?

- A) No — transactions only exist after `BEGIN`.
- B) Yes — it runs in a one-statement autocommit transaction.
- C) Only if the table has a primary key.
- D) Only in SERIALIZABLE mode.

---

**Q3.** In PostgreSQL, after a statement raises an error inside a transaction, the next statement fails with "current transaction is aborted." What must you do?

- A) Nothing; the next statement will retry automatically.
- B) `COMMIT` to save the work done so far.
- C) `ROLLBACK` (or `ROLLBACK TO SAVEPOINT`) to leave the aborted state.
- D) Restart the PostgreSQL server.

---

**Q4.** `ROLLBACK TO SAVEPOINT s` does what?

- A) Ends the transaction and discards everything.
- B) Commits everything up to `s`.
- C) Discards work done after `s`, keeps work before `s`, and keeps the transaction open.
- D) Deletes the savepoint but changes nothing else.

---

**Q5.** Which anomaly is defined as "reading the same row twice in one transaction and getting different values"?

- A) Dirty read
- B) Non-repeatable read
- C) Phantom read
- D) Write skew

---

**Q6.** Which anomaly can you **never** reproduce in PostgreSQL, at any isolation level?

- A) Non-repeatable read
- B) Phantom read
- C) Dirty read
- D) Write skew

---

**Q7.** At PostgreSQL's `REPEATABLE READ`, which anomaly is still possible?

- A) Dirty read
- B) Non-repeatable read
- C) Phantom read
- D) Serialization anomaly (write skew)

---

**Q8.** Two transactions at REPEATABLE READ each read "2 doctors on call" and each set a *different* doctor off-call; both commit; now zero are on call. What is this, and what would have prevented it?

*(Short answer: name the anomaly and the isolation level that catches it.)*

---

**Q9.** When a `SERIALIZABLE` transaction fails at `COMMIT` with SQLSTATE `40001`, whose responsibility is it to recover, and how?

- A) The database retries it automatically.
- B) The application must catch the error and retry the whole transaction.
- C) You must lower the isolation level and rerun.
- D) The row is permanently locked until a DBA intervenes.

---

**Q10.** In PostgreSQL MVCC, what does an `UPDATE` do to the existing row version?

- A) Overwrites it in place.
- B) Leaves the old version and writes a new version, marking the old one's `xmax`.
- C) Deletes it immediately and frees the space.
- D) Locks the whole table until commit.

---

**Q11.** The system column `xmin` on a tuple records:

- A) The physical location of the row.
- B) The transaction id that created that row version.
- C) The number of times the row was updated.
- D) The transaction id that deleted the row.

---

**Q12.** Why does PostgreSQL need `VACUUM`? Give the one-sentence reason, and name one problem that follows from a table that never gets vacuumed.

*(Short answer.)*

---

**Q13.** Which clause makes a `SELECT` lock the returned rows so a concurrent session's matching `SELECT … FOR UPDATE` blocks until you commit?

- A) `FOR SHARE`
- B) `FOR UPDATE`
- C) `READ ONLY`
- D) `SKIP LOCKED`

---

**Q14.** Session A locks row 1 then waits for row 2; Session B locks row 2 then waits for row 1. What happens, roughly how long until it happens, and what single discipline prevents this class of bug?

*(Short answer: name the outcome, the default timeout, and the fix.)*

---

## Answer key

<details>
<summary>Reveal after attempting all fourteen</summary>

1. **B** — atomicity is all-or-nothing. (A is durability, C is isolation, D is consistency.)
2. **B** — every statement runs in a transaction; without `BEGIN` it's a one-statement autocommit transaction that commits the instant it finishes.
3. **C** — a Postgres transaction is poisoned after any error; only `ROLLBACK` (or `ROLLBACK TO SAVEPOINT`) leaves the aborted state. A `COMMIT` is turned into a rollback.
4. **C** — it rewinds to the marker, discarding later work, but keeps the transaction alive and un-aborted.
5. **B** — non-repeatable read (a single *row's value* changed). Contrast the phantom, where the *set of rows* changed.
6. **C** — dirty read. Postgres never shows uncommitted data; `READ UNCOMMITTED` behaves as `READ COMMITTED`, because MVCC visibility requires the creating transaction to have committed.
7. **D** — write skew / serialization anomaly. Postgres's REPEATABLE READ (snapshot isolation) already forbids dirty, non-repeatable, and phantom reads; only serialization anomalies remain, and only SERIALIZABLE catches them.
8. **Write skew** (a serialization anomaly). Each transaction was individually valid but their combination broke the invariant. **SERIALIZABLE** catches it — one of the two commits fails with `40001`.
9. **B** — the application must catch `40001` and retry the whole transaction with a fresh snapshot. The database will not retry for you.
10. **B** — MVCC never overwrites; it writes a new tuple version and marks the old one's `xmax`. `VACUUM` later reclaims the dead old version.
11. **B** — `xmin` is the creating transaction's id. (`xmax` is the deleting/superseding xid; `ctid` is the physical location.)
12. Because MVCC leaves dead tuples behind on every `UPDATE`/`DELETE`, `VACUUM` reclaims them. A never-vacuumed table suffers **bloat** (grows far larger than its live data) and risks **transaction-id wraparound** (a protective read-only shutdown). Either is acceptable full marks.
13. **B** — `FOR UPDATE` takes the strongest row lock; a second `FOR UPDATE` on the same row blocks until you commit. (`SKIP LOCKED` would skip it instead of waiting.)
14. A **deadlock**. Postgres detects the wait cycle after `deadlock_timeout` (default **1 second**) and aborts one transaction (`40P01`). The fix: **acquire locks in a consistent order** (e.g., always ascending `id`) so a cycle can't form — and still wrap the transaction in a retry loop for the conflicts that slip through.

</details>

**Scoring:** 12–14 → move to Week 6. 9–11 → re-read the lecture behind each miss. <9 → re-work Lectures 2 and 3 and redo Exercises 2 and 3 before continuing.
