# Week 8 — Homework

Six practice problems, ~4 hours total. These reinforce the lectures with fresh data so you can't lean on memorized answers from the exercises. Do them in `psql` or `sqlite3`, save your queries, and check each against the stated expectation.

Shared seed for problems 1–4 (create once):

```sql
CREATE TABLE txns (
  id        INTEGER PRIMARY KEY,
  account   TEXT,
  ts        DATE,
  amount    INTEGER          -- positive = deposit, negative = withdrawal
);

INSERT INTO txns (id, account, ts, amount) VALUES
  (1,'A','2026-07-01', 100), (2,'A','2026-07-02', -30), (3,'A','2026-07-03',  50),
  (4,'A','2026-07-05', -20), (5,'A','2026-07-06', -20),
  (6,'B','2026-07-01', 200), (7,'B','2026-07-02',  40), (8,'B','2026-07-04', -60),
  (9,'B','2026-07-05',  10);
```

---

## Problem 1 — Account balance over time (running total per partition)

For each transaction, show a `balance` column: the running sum of `amount` **per account**, ordered by `ts`. Account A should end at `80`; account B at `190`.

*Skills: `SUM() OVER (PARTITION BY … ORDER BY … ROWS …)`.*

**Check:** A's balances are `100, 70, 120, 100, 80`; B's are `200, 240, 180, 190`.

---

## Problem 2 — Rank each account's transactions by size

Add a column `rk` ranking each transaction **within its account** by absolute `amount`, largest first, using `DENSE_RANK`. Then return only each account's single largest-magnitude transaction (filter in an outer query).

*Skills: `DENSE_RANK`, `ABS`, top-1-per-group.*

**Check:** A's biggest is `id 1` (100); B's is `id 1` (200).

---

## Problem 3 — Day-over-day balance change

Using your Problem 1 balance, add `delta` = today's balance − yesterday's balance **for the same account** (via `LAG` over the balance). The first row per account is `NULL`. Explain in a comment why this equals the current row's `amount` (it should — prove it to yourself).

*Skills: `LAG`, partitioned offsets.*

---

## Problem 4 — Deposit vs withdrawal pivot

Per account, produce two columns: `total_deposits` (sum of positive amounts) and `total_withdrawals` (sum of negative amounts), using `FILTER` (PostgreSQL / SQLite 3.30+) or `CASE`. One row per account.

*Skills: conditional aggregation / pivot.*

**Check:** A → deposits `150`, withdrawals `-70`. B → deposits `250`, withdrawals `-60`.

---

## Problem 5 — Streaks of positive days

New seed:

```sql
CREATE TABLE days (d DATE, net INTEGER);
INSERT INTO days VALUES
  ('2026-08-01',  5), ('2026-08-02', 3), ('2026-08-03', -2),
  ('2026-08-04',  8), ('2026-08-05', 1), ('2026-08-06',  4),
  ('2026-08-07', -1), ('2026-08-08', 2);
```

Find the **longest run of consecutive days with `net > 0`**. Filter to positive days first, then apply the gaps-and-islands row_number-difference trick. The answer is the run `2026-08-04 … 2026-08-06` (length 3).

*Skills: gaps-and-islands on a filtered set.*

---

## Problem 6 — Percentile buckets

Using the `txns` table, assign every transaction to a **quartile** by `amount` (across all accounts, signed value ascending) with `NTILE(4)`, and also show its `PERCENT_RANK()`. Then write one sentence: what does a `PERCENT_RANK` of `0` mean, and which transaction(s) have it?

*Skills: `NTILE`, `PERCENT_RANK`.*

---

## Submission

Commit a single `homework.sql` (all six problems, each under a comment header) plus your pasted outputs under `c33-week-08/homework/`. Tick the checks above as you go.

## Self-assessment

- [ ] Every running total is partitioned correctly (accounts never mix).
- [ ] You used an explicit `ROWS` frame for the running balances and can say why.
- [ ] Your pivot totals match the checks.
- [ ] Problem 5's longest streak is length 3, dates `08-04..08-06`.
- [ ] You can explain `PERCENT_RANK = 0` in one sentence.
