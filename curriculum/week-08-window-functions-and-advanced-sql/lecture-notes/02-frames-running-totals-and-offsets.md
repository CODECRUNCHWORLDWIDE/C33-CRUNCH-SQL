# Lecture 2 — Frames, Running Totals & Offset Functions

> **Duration:** ~2 hours. **Outcome:** You can state the *default frame* and explain why adding `ORDER BY` to an aggregate window silently turns it into a running total; write `ROWS`/`RANGE`/`GROUPS` frames deliberately; and use `LAG`, `LEAD`, `FIRST_VALUE`, and `LAST_VALUE` to reach neighboring rows for period-over-period math.

## 1. The frame: the third part of `OVER`

Lecture 1 covered `PARTITION BY` and `ORDER BY`. The third piece is the **frame** — the subset of the partition that a window *aggregate* actually sees for the current row. Ranking functions ignore the frame; aggregate window functions (`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`) and some offset functions (`FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`) obey it.

A frame is written after the `ORDER BY`:

```
<mode> BETWEEN <start> AND <end> [EXCLUDE …]
```

where `<mode>` is `ROWS`, `RANGE`, or `GROUPS`, and start/end are chosen from:

| Boundary | Meaning |
|----------|---------|
| `UNBOUNDED PRECEDING` | The first row of the partition |
| `n PRECEDING` | `n` rows (or values, in `RANGE`) before the current row |
| `CURRENT ROW` | The current row |
| `n FOLLOWING` | `n` rows/values after the current row |
| `UNBOUNDED FOLLOWING` | The last row of the partition |

## 2. The default frame — the bug that bites everyone

Here is the single most important, most misunderstood fact about window functions.

**If you write `ORDER BY` inside `OVER` but omit an explicit frame, the default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.**

That means the aggregate accumulates from the start of the partition up to the current row — a **running total**, not the partition total. So these two queries give *different* answers:

```sql
-- No ORDER BY: frame = whole partition. Same total on every row.
SELECT id, category, amount,
       SUM(amount) OVER (PARTITION BY category) AS category_total
FROM sales;

-- ORDER BY present: default frame = start..current row. This is a RUNNING total.
SELECT id, category, amount,
       SUM(amount) OVER (PARTITION BY category ORDER BY id) AS running_total
FROM sales;
```

People add `ORDER BY` "just to be tidy," expecting the total to stay the same, and are baffled when their totals start climbing row by row. Now you know: **`ORDER BY` in an aggregate window turns it into a running aggregate.** If you want the full partition total *and* an order, set the frame explicitly:

```sql
SUM(amount) OVER (PARTITION BY category ORDER BY id
                  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
```

## 3. `ROWS` vs `RANGE` vs `GROUPS`

All three describe a frame, but they count differently. Suppose the current row's `ORDER BY` value has ties.

| Mode | Counts by | `n PRECEDING` means | Ties (peers) |
|------|-----------|---------------------|--------------|
| `ROWS` | Physical rows | Exactly `n` rows back | Each tied row is separate |
| `RANGE` | `ORDER BY` **values** | All rows whose value is within `n` of the current value | All peers included together |
| `GROUPS` | Distinct-value **groups** | `n` peer-groups back | All peers in a group counted together |

The practical upshot:

- **`ROWS`** is what you want for "the last 3 rows," "a 7-row moving average," anything counted in *records*. It's precise and predictable. Use it by default.
- **`RANGE`** with `CURRENT ROW` includes **all rows tied with the current row** in the frame — the default frame is `RANGE`, so a running total over a column with duplicate `ORDER BY` values jumps in steps, giving *every tied row the same cumulative value*. This surprises people. If you want strict row-by-row accumulation even across ties, say `ROWS`.
- **`RANGE`** also supports value offsets — `RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW` over a date column gives a true "trailing 7 calendar days" window, correctly handling gaps where some days have no rows. (PostgreSQL supports interval range offsets; SQLite supports numeric `RANGE` offsets but not `INTERVAL` — use `ROWS` or a numeric epoch there.)
- **`GROUPS`** (PostgreSQL 11+, SQLite 3.28+) counts distinct peer-groups; niche but occasionally exactly right for "the last 3 *distinct dates*."

### Tie demonstration — `ROWS` vs `RANGE`

```sql
CREATE TABLE t (id INT, d DATE, amount INT);
INSERT INTO t VALUES
  (1, '2026-01-01', 10),
  (2, '2026-01-01', 20),   -- same date as row 1
  (3, '2026-01-02', 30);

SELECT id, d, amount,
       SUM(amount) OVER (ORDER BY d ROWS  BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS by_rows,
       SUM(amount) OVER (ORDER BY d RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS by_range
FROM t;
```

```
 id |     d      | amount | by_rows | by_range
----+------------+--------+---------+---------
  1 | 2026-01-01 |   10   |   10    |   30      <- RANGE includes the tied 2026-01-01 row
  2 | 2026-01-01 |   20   |   30    |   30
  3 | 2026-01-02 |   30   |   60    |   60
```

`by_range` shows `30` on *both* Jan-1 rows because `RANGE` treats equal-date rows as one peer group. `by_rows` accumulates strictly one row at a time. This is why **`ROWS` is the safer default** for running totals.

## 4. Running totals

The canonical running total:

```sql
SELECT order_date, amount,
       SUM(amount) OVER (ORDER BY order_date, id
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM orders;
```

Add a tiebreaker (`, id`) so rows on the same date accumulate deterministically. Partition it to get a *per-group* running total:

```sql
SUM(amount) OVER (PARTITION BY customer_id ORDER BY order_date, id
                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

## 5. Moving averages

A moving average is a *bounded* frame — the last N rows including the current one:

```sql
-- 7-row trailing moving average
SELECT day, revenue,
       AVG(revenue) OVER (ORDER BY day
                          ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7
FROM daily_revenue;
```

`6 PRECEDING AND CURRENT ROW` = 7 rows total. A **centered** 5-row average uses `ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING`. Note that the first few rows have a shorter frame (there aren't 6 rows before them yet), so early values are averages of fewer points — expected, not a bug. If you need a *calendar* 7-day window that tolerates missing days, use a date `RANGE` frame instead of `ROWS` (see §3).

## 6. Offset functions: `LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE`

These fetch a value from *another* row relative to the current one — the key to period-over-period comparisons.

| Function | Returns |
|----------|---------|
| `LAG(expr, offset, default)` | `expr` from `offset` rows **before** the current row (default `offset` 1) |
| `LEAD(expr, offset, default)` | `expr` from `offset` rows **after** the current row |
| `FIRST_VALUE(expr)` | `expr` from the **first** row of the frame |
| `LAST_VALUE(expr)` | `expr` from the **last** row of the frame |
| `NTH_VALUE(expr, n)` | `expr` from the **n-th** row of the frame |

`LAG`/`LEAD` ignore the frame — they navigate by position within the ordered partition. `FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE` **do** respect the frame, which leads to the most famous window-function gotcha (next section).

### Period-over-period with `LAG`

```sql
SELECT month, revenue,
       LAG(revenue) OVER (ORDER BY month) AS prev_month,
       revenue - LAG(revenue) OVER (ORDER BY month) AS delta,
       ROUND( 100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
              / NULLIF(LAG(revenue) OVER (ORDER BY month), 0), 1) AS pct_growth
FROM monthly_revenue;
```

`NULLIF(prev, 0)` guards against divide-by-zero. The third argument to `LAG`/`LEAD` supplies a **default** when there is no such row (e.g. the first row has no previous): `LAG(revenue, 1, 0)` returns `0` instead of `NULL` for month one.

### The `LAST_VALUE` trap

Intuitively, `LAST_VALUE(x) OVER (ORDER BY d)` should give the last value in the partition. It usually gives the **current row's** value instead — because the default frame ends at `CURRENT ROW`, so "the last row of the frame" *is* the current row. Fix it by extending the frame to the end of the partition:

```sql
-- WRONG: returns the current row's value, not the partition's last
LAST_VALUE(price) OVER (PARTITION BY sym ORDER BY d)

-- RIGHT: frame runs to the end of the partition
LAST_VALUE(price) OVER (PARTITION BY sym ORDER BY d
                        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
```

`FIRST_VALUE` doesn't have this problem because the default frame *starts* at `UNBOUNDED PRECEDING`. Many people avoid `LAST_VALUE` entirely and instead use `FIRST_VALUE(...) OVER (... ORDER BY d DESC)`.

## 7. A worked example — a stock-price mini-report

```sql
CREATE TABLE prices (sym TEXT, d DATE, close NUMERIC);
INSERT INTO prices VALUES
  ('ACME','2026-01-02', 100),
  ('ACME','2026-01-03', 104),
  ('ACME','2026-01-04',  99),
  ('ACME','2026-01-05', 110),
  ('ACME','2026-01-06', 112);

SELECT d, close,
       close - LAG(close) OVER w                          AS day_change,
       ROUND(AVG(close) OVER (PARTITION BY sym ORDER BY d
              ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS ma3,
       FIRST_VALUE(close) OVER w                          AS first_close,
       LAST_VALUE(close)  OVER (PARTITION BY sym ORDER BY d
              ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_close
FROM prices
WINDOW w AS (PARTITION BY sym ORDER BY d);
```

Expected:

```
     d      | close | day_change |  ma3   | first_close | last_close
------------+-------+------------+--------+-------------+-----------
 2026-01-02 |  100  |            | 100.00 |     100     |    112
 2026-01-03 |  104  |     4      | 102.00 |     100     |    112
 2026-01-04 |   99  |    -5      | 101.00 |     100     |    112
 2026-01-05 |  110  |    11      | 104.33 |     100     |    112
 2026-01-06 |  112  |     2      | 107.00 |     100     |    112
```

`day_change` is blank (NULL) on the first row — no previous. `ma3` is the 3-day trailing average (first two rows use fewer points). `last_close` correctly returns `112` on every row because we extended the frame.

## 8. Check yourself

- What is the default frame when `ORDER BY` is present, and how does it differ from no `ORDER BY` at all?
- Why can a running total over a column with duplicate `ORDER BY` values look "stair-stepped" under the default frame, and how does `ROWS` fix it?
- Write the frame clause for a 7-row trailing moving average.
- Why does `LAST_VALUE` usually return the current row's value, and what's the fix?
- What does the third argument to `LAG` do?
- When would you choose `RANGE` over `ROWS`?

Answer all six, then move to Lecture 3 — where these primitives combine into gaps-and-islands and sessionization.

## Further reading

- **PostgreSQL 16 — Window Function Calls (frames, `ROWS`/`RANGE`/`GROUPS`, `EXCLUDE`):** <https://www.postgresql.org/docs/16/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS>
- **PostgreSQL 16 — `lag`, `lead`, `first_value`, `last_value`:** <https://www.postgresql.org/docs/16/functions-window.html>
- **SQLite — The frame specification & offset functions:** <https://www.sqlite.org/windowfunctions.html#frame_specifications>
