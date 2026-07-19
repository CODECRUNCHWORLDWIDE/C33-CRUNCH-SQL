# Exercise 3 — Fix a Mis-Estimated Plan

**Goal:** Deliberately break the planner's estimates, watch it pick a bad plan, then fix the *estimate* (not the query) and watch the plan correct itself. This is the single most common real-world tuning fix.

**Estimated time:** 40 minutes.

## Setup

Dataset from [`exercises/README.md`](./README.md). We'll create correlation the planner doesn't know about, and staleness it can't see.

## Part A — Stale statistics

### 1. Establish a good baseline

```sql
ANALYZE orders;
EXPLAIN (ANALYZE) SELECT * FROM orders WHERE created_at >= '2025-01-01';
```

Note the plan and the estimated-vs-actual rows. It should be close.

### 2. Break it — bulk-insert without re-analyzing

```sql
INSERT INTO orders (id, customer_id, status, total, created_at)
SELECT 3000000 + g, 1 + (g % 100000), 'shipped', 10.00,
       timestamptz '2026-01-01' + (g % 200) * interval '1 day'
FROM generate_series(1, 2000000) g;
-- Do NOT run ANALYZE yet.
```

You just added 2,000,000 rows in a date range the stats know nothing about.

### 3. Observe the damage

```sql
EXPLAIN (ANALYZE) SELECT * FROM orders WHERE created_at >= '2026-01-01';
```

- Record the **estimated** rows vs the **actual** rows. How far off is it?
- Did the scan or join choice change from a sensible one?

### 4. Fix the estimate

```sql
ANALYZE orders;
EXPLAIN (ANALYZE) SELECT * FROM orders WHERE created_at >= '2026-01-01';
```

- Record the new estimated-vs-actual. Did the plan change? Did time improve?

Write two sentences: *what the wrong estimate caused, and what one command fixed it.*

## Part B — Correlated columns

### 1. See the independence assumption fail

`city` and `country` in our data are perfectly correlated (each city belongs to exactly one country).

```sql
EXPLAIN (ANALYZE)
SELECT * FROM customers WHERE city = 'Paris' AND country = 'FR';
```

- Record estimated rows vs actual rows. The estimate should be *far too low* — the planner multiplied two selectivities as if independent.

### 2. Teach the planner the correlation

```sql
CREATE STATISTICS cust_city_country (dependencies, ndistinct)
  ON city, country FROM customers;
ANALYZE customers;
```

### 3. Confirm the fix

```sql
EXPLAIN (ANALYZE)
SELECT * FROM customers WHERE city = 'Paris' AND country = 'FR';
```

- Record the new estimate. Is it now close to actual?
- Write one sentence explaining what `CREATE STATISTICS` told the planner.

## Acceptance criteria

- [ ] `solutions.md` shows the before/after estimated-vs-actual rows for both parts.
- [ ] Part A demonstrates a large estimate error *before* `ANALYZE` and a corrected one *after*.
- [ ] Part B shows the multi-column estimate improving after `CREATE STATISTICS`.
- [ ] You wrote the two required one/two-sentence explanations.

## Stretch

- In Part A, use `pg_stat_user_tables` to show `last_analyze` before and after your manual `ANALYZE`.
- Add `mcv` to the extended statistics object (`... (dependencies, ndistinct, mcv) ...`) and re-analyze. Does the estimate get even closer for a specific common combination?
- Clean up: `DELETE FROM orders WHERE id > 3000000; ANALYZE orders;` and confirm the estimate returns to baseline.

## Hints

<details>
<summary>Where to see the current estimate quickly</summary>

You don't always need `ANALYZE` (which runs the query). Plain `EXPLAIN SELECT ...` shows the *estimate* instantly and safely. Use it to iterate fast, then confirm with `EXPLAIN (ANALYZE)` once.

</details>

## Submission

Commit `solutions.md` under `c33-week-07/exercises/exercise-03/`.
