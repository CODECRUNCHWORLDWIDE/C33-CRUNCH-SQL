# Exercise 3 — Gaps and Islands

**Goal:** Collapse consecutive values into runs ("islands") and find the holes between them ("gaps"), using the row_number-difference trick and `LEAD`.

**Estimated time:** 2 hours.

## Setup

A login log: which days each user opened the app. Some streaks, some gaps.

```sql
CREATE TABLE logins (
  user_id     INTEGER,
  login_date  DATE
);

INSERT INTO logins (user_id, login_date) VALUES
  -- Ana: 1,2,3, gap, 6,7, gap, 10
  (1, '2026-05-01'), (1, '2026-05-02'), (1, '2026-05-03'),
  (1, '2026-05-06'), (1, '2026-05-07'), (1, '2026-05-10'),
  -- Ben: 2,3,4,5, gap, 8
  (2, '2026-05-02'), (2, '2026-05-03'), (2, '2026-05-04'),
  (2, '2026-05-05'), (2, '2026-05-08');
```

## Tasks

Write each into `exercise-03.sql`.

### 1. Warm-up: see the trick

For **user 1 only**, list each `login_date` with its `ROW_NUMBER()` and the group key `login_date − ROW_NUMBER()`. Confirm the group key is constant within each run and changes at every gap.

- PostgreSQL: `login_date - (ROW_NUMBER() OVER (ORDER BY login_date))::int AS grp`
- SQLite: `julianday(login_date) - ROW_NUMBER() OVER (ORDER BY login_date) AS grp`

You should see three distinct `grp` values for user 1 (runs `1–3`, `6–7`, `10`).

### 2. Collapse each user's logins into streaks

For **both users**, produce one row per consecutive run: `user_id`, `streak_start`, `streak_end`, `streak_days`. Partition the `ROW_NUMBER()` by `user_id` and group by `user_id, grp`.

**Expected result:**

```
 user_id | streak_start | streak_end | streak_days
---------+--------------+------------+------------
    1    | 2026-05-01   | 2026-05-03 |     3
    1    | 2026-05-06   | 2026-05-07 |     2
    1    | 2026-05-10   | 2026-05-10 |     1
    2    | 2026-05-02   | 2026-05-05 |     4
    2    | 2026-05-08   | 2026-05-08 |     1
```

### 3. Each user's longest streak

Wrap task 2 in a CTE and return, per user, the row with the maximum `streak_days` (ties: return all). Ana's longest is 3 days; Ben's is 4.

### 4. Find the gaps directly with `LEAD`

For **user 1**, use `LEAD(login_date)` to find missing ranges: where the next login is more than one day after the current, report `gap_start` (current + 1 day) and `gap_end` (next − 1 day).

**Expected result** (user 1):

```
 gap_start  |  gap_end
------------+-----------
 2026-05-04 | 2026-05-05
 2026-05-08 | 2026-05-09
```

- PostgreSQL: `login_date + 1` and `next_date - 1` (date + integer).
- SQLite: `date(login_date, '+1 day')` and `date(next_date, '-1 day')`; test the gap with `julianday(next_date) - julianday(login_date) > 1`.

### 5. Current streak length

For each user, compute the length of their **most recent** streak (the run containing their latest login). Ana's latest login is `2026-05-10` (a 1-day run); Ben's is `2026-05-08` (a 1-day run).

## Done when…

- [ ] `exercise-03.sql` has all five queries.
- [ ] Task 2's output matches the expected five rows exactly.
- [ ] Task 4 reports gaps `05-04..05-05` and `05-08..05-09` for user 1.
- [ ] You can explain, in one sentence, why `login_date − ROW_NUMBER()` is constant across a run of consecutive dates.
- [ ] You handled the `user_id` partition so users' streaks never bleed into each other.

## Stretch

- Add a `is_current` boolean to task 2 marking each user's most recent streak.
- Generalize the gap definition: treat a gap of **2 or fewer** days as "still the same streak" (a grace period). Hint: this changes the grouping logic — you can't use the plain row_number trick; measure the gap with `LAG`, flag breaks over the threshold, and running-`SUM` the flags (the sessionization pattern from Lecture 3).
- Count, per user, how many *distinct* streaks they had in May.

## Hints

<details>
<summary>The streak-collapse query</summary>

```sql
WITH numbered AS (
  SELECT user_id, login_date,
         login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date))::int AS grp
  FROM logins
)
SELECT user_id,
       MIN(login_date) AS streak_start,
       MAX(login_date) AS streak_end,
       COUNT(*)        AS streak_days
FROM numbered
GROUP BY user_id, grp
ORDER BY user_id, streak_start;
```

In SQLite, replace the `grp` expression with `julianday(login_date) - ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date)`.
</details>

## Submission

Commit `exercise-03.sql` under `c33-week-08/exercises/`.
