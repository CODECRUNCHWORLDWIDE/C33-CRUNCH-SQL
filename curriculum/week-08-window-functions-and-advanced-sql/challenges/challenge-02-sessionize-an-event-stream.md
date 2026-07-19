# Challenge 2 — Sessionize an Event Stream

**Time:** ~2 hours. **Difficulty:** Hard.

## Scenario

Your app writes one row per user action to an `events` table — clicks, page views, API calls, a firehose. Product wants **sessions**: a session is a burst of activity by one user, and it *ends* when that user goes quiet for **30 minutes or more**. The next event after a 30-minute lull starts a fresh session. You need, per session: the user, a session number, its start and end time, its duration, and how many events it contained.

This is the pattern from Lecture 3 (`LAG` → flag → running `SUM`), taken end to end on messier data.

## Seed data

```sql
CREATE TABLE events (
  id          INTEGER PRIMARY KEY,
  user_id     INTEGER,
  event_time  TIMESTAMP
);

INSERT INTO events (id, user_id, event_time) VALUES
  -- User 1: two sessions (a >30-min gap between 10:20 and 11:05)
  (1, 1, '2026-06-01 10:00'),
  (2, 1, '2026-06-01 10:10'),
  (3, 1, '2026-06-01 10:20'),
  (4, 1, '2026-06-01 11:05'),   -- 45-min gap → new session
  (5, 1, '2026-06-01 11:15'),
  -- User 2: one long session (every gap < 30 min) then a lone event next day
  (6, 2, '2026-06-01 09:00'),
  (7, 2, '2026-06-01 09:25'),
  (8, 2, '2026-06-01 09:50'),
  (9, 2, '2026-06-02 09:00'),   -- huge gap → new session
  -- User 3: a single event (edge case: session of length 1, zero duration)
  (10, 3, '2026-06-01 14:00');
```

## Your task

Produce one row per session:

```
 user_id | session_no |    session_start    |     session_end     | duration_min | n_events
---------+------------+---------------------+---------------------+--------------+---------
    1    |     1      | 2026-06-01 10:00    | 2026-06-01 10:20    |      20      |    3
    1    |     2      | 2026-06-01 11:05    | 2026-06-01 11:15    |      10      |    2
    2    |     1      | 2026-06-01 09:00    | 2026-06-01 09:50    |      50      |    3
    2    |     2      | 2026-06-02 09:00    | 2026-06-02 09:00    |       0      |    1
    3    |     1      | 2026-06-01 14:00    | 2026-06-01 14:00    |       0      |    1
```

`session_no` restarts at 1 for each user. `duration_min` is `session_end − session_start` in minutes (a single-event session is 0).

## Constraints

- One query, CTEs allowed. Pure SQL.
- The 30-minute threshold is a boundary case: a gap of **exactly** 30 minutes — decide and document whether that starts a new session. State your choice (this challenge treats `>= 30 min` as a new session, i.e. strictly less than 30 stays in the same session).
- Sessions must **never** cross users. Partition every window by `user_id`.
- Handle the single-event user (user 3) and the first event of every user (no previous event → session start).
- Target PostgreSQL 16. SQLite swaps below.

## Hints

<details>
<summary>The three-step recipe</summary>

```sql
WITH gaps AS (
  SELECT id, user_id, event_time,
         event_time - LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS since_prev
  FROM events
),
flagged AS (
  SELECT *,
         CASE WHEN since_prev IS NULL OR since_prev >= INTERVAL '30 minutes'
              THEN 1 ELSE 0 END AS is_start
  FROM gaps
),
numbered AS (
  SELECT *,
         SUM(is_start) OVER (PARTITION BY user_id ORDER BY event_time
                             ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_no
  FROM flagged
)
SELECT user_id, session_no,
       MIN(event_time) AS session_start,
       MAX(event_time) AS session_end,
       EXTRACT(EPOCH FROM (MAX(event_time) - MIN(event_time))) / 60 AS duration_min,
       COUNT(*) AS n_events
FROM numbered
GROUP BY user_id, session_no
ORDER BY user_id, session_no;
```
</details>

<details>
<summary>SQLite version</summary>

SQLite has no `INTERVAL`. Work in seconds via `julianday` (which returns days as a float): `(julianday(event_time) - julianday(LAG(event_time) OVER (...))) * 24 * 60` gives minutes. Flag a start when that is `NULL` or `>= 30`. Duration: `(julianday(MAX(event_time)) - julianday(MIN(event_time))) * 24 * 60`.
</details>

## How success is judged

| Criterion | What "great" looks like |
|-----------|-------------------------|
| Correctness | Output matches the expected five sessions exactly |
| Boundary handling | The 30-min rule is applied consistently and documented |
| No cross-user bleed | Every window partitions by `user_id`; user 3's lone event is its own session |
| Edge cases | Single-event and first-event rows handled (session start, duration 0) |
| Clarity | The `LAG` → flag → running-`SUM` structure is legible in named CTEs |

## Stretch

- Add **average events per session** and **average session duration** per user (aggregate over your session rows).
- Make the inactivity threshold a parameter and show results for 15-, 30-, and 60-minute windows. How does session count change?
- Add a `session_id` that is **globally unique** across all users (not just per-user), e.g. `user_id` concatenated with `session_no`, or a `DENSE_RANK` over `(user_id, session_no)`.
- Detect and report **overlapping / out-of-order** timestamps if any exist (they don't in this seed — add one and make your query robust to it).

## Submission

Commit `challenge-02.sql` + `NOTES.md` (document your 30-min boundary decision) under `c33-week-08/challenges/`.
