# Exercise 2 — Running Totals & `LAG`/`LEAD`

**Goal:** Build a cumulative total, a moving average, and period-over-period growth — the three columns every "revenue over time" dashboard needs.

**Estimated time:** 2 hours.

## Setup

Two weeks of daily revenue for a small shop. (PostgreSQL uses real `DATE`s; SQLite stores them as `TEXT` in ISO format, which sorts correctly.)

```sql
CREATE TABLE daily_revenue (
  day      DATE PRIMARY KEY,
  revenue  INTEGER
);

INSERT INTO daily_revenue (day, revenue) VALUES
  ('2026-03-01', 100),
  ('2026-03-02', 120),
  ('2026-03-03',  90),
  ('2026-03-04', 140),
  ('2026-03-05', 160),
  ('2026-03-06', 130),
  ('2026-03-07', 180),
  ('2026-03-08', 200),
  ('2026-03-09', 150),
  ('2026-03-10', 170);
```

## Tasks

Write each into `exercise-02.sql`.

### 1. Running total of revenue

Add a `running_total` column that accumulates from the first day to the current day. Use an explicit `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` frame so ties (there are none here, but build the habit) accumulate one row at a time.

**Expected result:**

```
    day     | revenue | running_total
------------+---------+--------------
 2026-03-01 |   100   |     100
 2026-03-02 |   120   |     220
 2026-03-03 |    90   |     310
 2026-03-04 |   140   |     450
 2026-03-05 |   160   |     610
 2026-03-06 |   130   |     740
 2026-03-07 |   180   |     920
 2026-03-08 |   200   |    1120
 2026-03-09 |   150   |    1270
 2026-03-10 |   170   |    1440
```

### 2. 3-day trailing moving average

Add `ma3 = AVG(revenue) OVER (ORDER BY day ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)`. Round to 1 decimal. The first two days average fewer points — that's expected.

**Expected `ma3`** (rounded): `100.0, 110.0, 103.3, 116.7, 130.0, 143.3, 156.7, 170.0, 176.7, 173.3`.

### 3. Day-over-day change with `LAG`

Add `prev_day` (yesterday's revenue via `LAG(revenue)`) and `change` (today − yesterday). The first row's `change` is `NULL`. Then add `pct_change` as a rounded percentage, guarding division with `NULLIF(prev, 0)`.

### 4. Best day so far

Add a `max_so_far` column: the highest single-day revenue from the start up to the current day (`MAX(revenue) OVER (ORDER BY day ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)`). Notice it's non-decreasing.

### 5. Look ahead with `LEAD`

Add `next_day` = tomorrow's revenue, and a text column `trend` that is `'up'` when tomorrow is higher, `'down'` when lower, `'flat'` when equal, and `'—'` on the last row (no tomorrow). Use `LEAD(revenue)` and a `CASE`.

## Done when…

- [ ] `exercise-02.sql` has all five tasks.
- [ ] `running_total`'s last value is `1440` and `max_so_far`'s last value is `200`.
- [ ] Your `ma3` matches the expected rounded values.
- [ ] Row 1's `change`, `pct_change`, and `prev_day` are `NULL`; the last row's `next_day`/`trend` handle the missing tomorrow.
- [ ] You can explain why `ROWS` (not `RANGE`) is the correct frame mode for tasks 1, 2, and 4.

## Stretch

- Compute a **centered** 3-day average (`ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING`) and compare it to the trailing one.
- Add a **7-day** trailing sum and a column for **percent of the two-week total** (`100.0 * revenue / SUM(revenue) OVER ()`).
- Reframe task 2 as a *calendar* 7-day window using a date `RANGE` frame (PostgreSQL: `RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW`). Insert a row with a missing day in between and observe how `RANGE` and `ROWS` diverge.

## Hints

<details>
<summary>The three core columns</summary>

```sql
SELECT day, revenue,
       SUM(revenue) OVER w_run                                     AS running_total,
       ROUND(AVG(revenue) OVER (ORDER BY day
              ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 1)        AS ma3,
       revenue - LAG(revenue) OVER (ORDER BY day)                  AS change
FROM daily_revenue
WINDOW w_run AS (ORDER BY day ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
ORDER BY day;
```
</details>

<details>
<summary>SQLite ROUND note</summary>

SQLite's `ROUND(x, 1)` behaves like PostgreSQL's for these values. If a moving average shows more decimals than you expect, you forgot the second argument to `ROUND`.
</details>

## Submission

Commit `exercise-02.sql` under `c33-week-08/exercises/`.
