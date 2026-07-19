# Mini-Project — Cohort + Running-Total Dashboard Query

> Build the one query a founder would screenshot in a board deck: a month-by-month view of the business that shows **new users**, **retained users**, **revenue**, a **running (cumulative) revenue** line, and **month-over-month growth** — all from a raw events-and-orders schema, all in SQL.

**Estimated time:** 6–7 hours, spread across Thursday–Saturday.

This mini-project ties the whole week together. You'll use CTEs (Week 3), cohort logic and `FILTER` pivots (Lecture 3), running totals and `LAG` growth (Lecture 2), and ranking to find your best month (Lecture 1). The deliverable is a **single, documented, runnable SQL file** that produces a monthly dashboard table — plus a short write-up reading the numbers like an analyst.

---

## The schema

Two tables. Create them and load the seed (this is your dataset — you may extend it, but keep these rows so your results are checkable):

```sql
CREATE TABLE users (
  user_id     INTEGER PRIMARY KEY,
  signup_time TIMESTAMP
);

CREATE TABLE orders (
  order_id    INTEGER PRIMARY KEY,
  user_id     INTEGER REFERENCES users(user_id),
  order_time  TIMESTAMP,
  amount      NUMERIC
);

INSERT INTO users (user_id, signup_time) VALUES
  (1,'2026-01-04'), (2,'2026-01-18'), (3,'2026-01-27'),
  (4,'2026-02-05'), (5,'2026-02-22'),
  (6,'2026-03-08'), (7,'2026-03-15'), (8,'2026-03-29'),
  (9,'2026-04-02'), (10,'2026-04-25');

INSERT INTO orders (order_id, user_id, order_time, amount) VALUES
  (1, 1,'2026-01-05', 40), (2, 2,'2026-01-19', 25), (3, 3,'2026-01-28', 60),
  (4, 1,'2026-02-06', 30), (5, 4,'2026-02-07', 80), (6, 2,'2026-02-20', 20),
  (7, 5,'2026-02-23', 50),
  (8, 1,'2026-03-03', 35), (9, 6,'2026-03-09', 70), (10, 7,'2026-03-16', 45),
  (11, 4,'2026-03-18', 65), (12, 8,'2026-03-30', 90),
  (13, 2,'2026-04-04', 20), (14, 9,'2026-04-05',120), (15, 6,'2026-04-12', 55),
  (16,10,'2026-04-26', 30), (17, 1,'2026-04-28', 40);
```

---

## Requirements

Your final query must return **one row per month** with at least these columns:

| Column | Definition |
|--------|-----------|
| `month` | The calendar month (`date_trunc('month', …)`) |
| `new_users` | Users whose **signup** month is this month |
| `active_users` | Distinct users who placed **an order** this month |
| `returning_users` | `active_users` who signed up in an **earlier** month |
| `revenue` | Total order `amount` this month |
| `running_revenue` | Cumulative revenue from the first month through this one |
| `mom_growth_pct` | Month-over-month revenue growth vs the previous month (`NULL` for month 1) |
| `pct_of_total` | This month's revenue as a share of all-time revenue |

Then, in the same file (a second query is fine), answer:

- **Which month had the highest revenue?** (ranking)
- **Cumulative revenue as a share of the annual total, per month** (running total ÷ grand total).

---

## Milestones

### Milestone 1 — Monthly revenue backbone (1h)

Get `month`, `revenue`, and `active_users` working with a plain `GROUP BY date_trunc('month', order_time)`. Verify the revenue per month by hand against the seed. Expected monthly revenue: **Jan 125, Feb 180, Mar 305, Apr 265** (add them: 875 total).

### Milestone 2 — Running total + growth (1.5h)

Add `running_revenue` (`SUM(revenue) OVER (ORDER BY month …)`), `mom_growth_pct` (`LAG`), and `pct_of_total` (`revenue / SUM(revenue) OVER ()`). This is where you *nest* a window over the `GROUP BY` result — `SUM(SUM(amount)) OVER (ORDER BY month)`. Expected `running_revenue`: **125, 305, 610, 875**.

### Milestone 3 — Users: new vs returning (2h)

Join in `users`. `new_users` = signups whose month = this month. `returning_users` = distinct users who ordered this month **and** whose signup month is strictly earlier. This needs a little care — build a per-order CTE that carries both the order month and the user's signup month, then aggregate with `FILTER`. Sanity check: in **February**, user 1 (signed up Jan) orders → returning; user 2 (Jan) orders → returning; user 4 (Feb) orders → new-and-active, not returning. So February `returning_users = 2`.

### Milestone 4 — Best-month + share queries (1h)

Add the two summary queries (highest-revenue month via `RANK`/`ROW_NUMBER`; cumulative share). 

### Milestone 5 — Write-up + polish (1h)

Write `dashboard-notes.md`: read the dashboard out loud in 5–8 sentences. Which month grew fastest? Is revenue accelerating? What share of revenue came from returning vs new users? Format the final query cleanly with named CTEs and a comment above each.

---

## Deliverables

A folder `c33-week-08/mini-project/` containing:

1. `dashboard.sql` — the full, runnable query (schema + seed + the dashboard query + the two summary queries), commented.
2. `output.txt` — the actual result tables you got, pasted from `psql`/`sqlite3`.
3. `dashboard-notes.md` — your 5–8 sentence analyst read of the numbers.

---

## Acceptance criteria

- [ ] `dashboard.sql` runs top-to-bottom on a fresh database with no errors (PostgreSQL 16 or SQLite).
- [ ] The monthly table has all eight required columns, one row per month, four rows total.
- [ ] Monthly revenue is `125, 180, 305, 265`; running revenue is `125, 305, 610, 875`.
- [ ] `mom_growth_pct` is `NULL` for January and correct thereafter (Feb = +44%).
- [ ] `returning_users` correctly excludes users signing up in the same month they first order.
- [ ] The best-month query names **March** (305) as the top revenue month.
- [ ] `dashboard-notes.md` reads the numbers, not just restates them.

---

## Rubric

| Criterion | Weight | "Great" looks like |
|-----------|------:|--------------------|
| Correctness | 30% | Every number matches the seed; spot-checkable by hand |
| Window mastery | 25% | Running total, `LAG` growth, and pct-of-total all correct, right frames |
| User logic | 20% | new vs returning is right, including the same-month edge case |
| Readability | 15% | Named CTEs, comments, a teammate could maintain it |
| Analysis | 10% | The write-up draws a real conclusion from the data |

---

## Why this matters

This *is* the query behind most early-stage "metrics dashboards" — before anyone buys a BI tool, someone writes exactly this. Being the person who can produce it from a raw schema, correctly, in one query, is a genuinely marketable skill. You'll reuse this shape in [C27 Crunch Data](../../../C27-CRUNCH-DATA/) and again in [C16 Crunch Pro Web Backend](../../../C16-CRUNCH-PRO-WEB-BACKEND/) when you wire it behind an API endpoint.

---

When done: push, then take the [Week 8 quiz](../quiz.md), and continue to [Week 9 — Procedures, Functions & Triggers](../../week-09-procedures-functions-and-triggers/).
