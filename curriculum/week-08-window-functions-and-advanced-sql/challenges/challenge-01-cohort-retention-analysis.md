# Challenge 1 — Cohort Retention Analysis

**Time:** ~2 hours. **Difficulty:** Hard.

## Scenario

You're the data person on a small product. Leadership wants the chart every growth team lives by: a **retention triangle**. Group users by the month they signed up (their *cohort*), then for each cohort measure what fraction were still active 1 month later, 2 months later, and so on. A healthy product shows the curve flattening (people who stay, stay); a leaky one drops to near zero by month 2.

You have a raw activity log. There is no separate "signups" table — a user's signup is simply their **first** event.

## Seed data

```sql
CREATE TABLE events (
  user_id     INTEGER,
  event_time  TIMESTAMP
);

INSERT INTO events (user_id, event_time) VALUES
  -- Cohort 2026-01: users 1,2,3
  (1, '2026-01-05 10:00'), (1, '2026-02-10 09:00'), (1, '2026-03-11 12:00'),
  (2, '2026-01-20 14:00'), (2, '2026-02-02 08:00'),
  (3, '2026-01-28 19:00'),
  -- Cohort 2026-02: users 4,5
  (4, '2026-02-03 11:00'), (4, '2026-03-04 10:00'), (4, '2026-04-06 15:00'),
  (5, '2026-02-15 16:00'), (5, '2026-04-19 09:00'),
  -- Cohort 2026-03: user 6
  (6, '2026-03-09 13:00'), (6, '2026-03-22 17:00'), (6, '2026-04-01 08:00');
```

## Your task

Produce a cohort-retention matrix: one row per signup-month cohort, columns for **month 0, 1, 2, 3** counting the number of distinct users from that cohort who were active in that relative month. Month 0 is the signup month itself (the cohort size). Then add a **percentage** view (each cell as a share of month 0).

Expected *counts* from the seed data (verify by hand — it's small on purpose):

```
 cohort  | m0 | m1 | m2 | m3
---------+----+----+----+----
 2026-01 |  3 |  2 |  1 |  0
 2026-02 |  2 |  1 |  2 |  0
 2026-03 |  1 |  1 |  0 |  0
```

Reading it: the January cohort has 3 users; 2 came back in month 1 (user 1 in Feb, user 2 in Feb), 1 in month 2 (user 1 in March), 0 in month 3. The percentage view turns row 1 into `100%, 67%, 33%, 0%`. Note the February cohort's month-2 count (2) is *higher* than its month-1 count (1) — a real "resurrection" effect (user 5 skipped March but returned in April). Retention curves are not always monotonic; don't "fix" that in your query.

## Constraints

- One query, CTEs allowed. No temp tables, no application code.
- `month_no` must be the count of **whole months between** signup month and activity month — not raw day differences. `date_trunc` to the month first, then compute the month delta.
- Count **distinct users** per cell (a user active twice in the same month counts once).
- Target PostgreSQL 16. On SQLite, replace `date_trunc('month', t)` with `strftime('%Y-%m', t)` and compute the month number from the year/month parts.

## Hints

<details>
<summary>Month-delta arithmetic (PostgreSQL)</summary>

Given `cohort_month` and `activity_month` (both month-truncated timestamps), the whole-month gap is:

```sql
(date_part('year',  activity_month) - date_part('year',  cohort_month)) * 12
+ (date_part('month', activity_month) - date_part('month', cohort_month))
```

`AGE()` also works: `EXTRACT(YEAR FROM age(...)) * 12 + EXTRACT(MONTH FROM age(...))`.
</details>

<details>
<summary>Structure</summary>

1. CTE `cohort`: `SELECT user_id, date_trunc('month', MIN(event_time)) AS cohort_month FROM events GROUP BY user_id`.
2. CTE `activity`: join events to `cohort`, compute `month_no` per event, and `date_trunc` the event month.
3. Final: `GROUP BY cohort_month`, and pivot with `COUNT(DISTINCT user_id) FILTER (WHERE month_no = k)` for k in 0..3.
</details>

<details>
<summary>Percentage view</summary>

Wrap the counts CTE and divide each column by `m0`: `ROUND(100.0 * m1 / NULLIF(m0,0))`. The `NULLIF` guards an empty cohort.
</details>

## How success is judged

| Criterion | What "great" looks like |
|-----------|-------------------------|
| Correctness | Counts match the expected matrix exactly; distinct-user counting is right |
| Month logic | Uses whole-month deltas via `date_trunc`, not day differences (off-by-one-proof) |
| Pivot | Clean `FILTER` (or `CASE`) aggregation; no repeated scans |
| Percentage view | Correct, with divide-by-zero guarded |
| Clarity | A reader follows cohort → activity → pivot without a diagram |

## Stretch

- Generalize to **N months** without hand-writing each `m0..mN` column — produce a *long* format (`cohort, month_no, retained, pct`) instead of a wide pivot, so it works for any horizon.
- Add a **cohort size** column and sort cohorts by it.
- Compute retention on a **weekly** grain instead of monthly and compare how noisy the curve gets.

## Submission

Commit `challenge-01.sql` + `NOTES.md` under `c33-week-08/challenges/`.
