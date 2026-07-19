# Challenge 2 — Diagnose an unused index

**Time:** ~60 minutes. **Difficulty:** Medium. **No single right answer.**

## Scenario

A teammate added an index last week "to speed up the customer lookup." It's still slow, and the index doesn't seem to be doing anything. Your job: figure out *why* the planner ignores an index that looks like it should apply — and this happens for **more than one reason** in this challenge.

## Setup

Create the situation on the `shop` database:

```sql
\c shop
-- teammate's index
CREATE INDEX ix_customers_email ON customers (email);
CREATE INDEX ix_orders_status ON orders (status);
ANALYZE customers;
ANALYZE orders;
```

## The four suspects

Below are four queries. Each *has* an index that could plausibly apply, yet the planner may still choose a `Seq Scan` — or use the index but not help. For **each**, capture the plan with `EXPLAIN (ANALYZE, BUFFERS)`, decide whether the index is used, and **explain the reason** in one or two sentences. Then, where it's fixable, fix it and re-measure.

```sql
-- Suspect 1
SELECT * FROM customers WHERE lower(email) = 'user42@example.com';

-- Suspect 2
SELECT * FROM orders WHERE status = 'paid';

-- Suspect 3
SELECT * FROM customers WHERE email LIKE 'user42%';

-- Suspect 4  (run right after a fresh bulk load, before ANALYZE — simulate it)
--   First: INSERT 200k rows, then query BEFORE running ANALYZE.
```

For Suspect 4, reproduce a stale-statistics plan:

```sql
INSERT INTO customers (email, country, created_at)
SELECT 'bulk' || g || '@example.com', 'US', now()
FROM generate_series(1, 200000) AS g;
-- DO NOT analyze yet
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM customers WHERE email = 'bulk150000@example.com';
-- then:
ANALYZE customers;
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM customers WHERE email = 'bulk150000@example.com';
```

## Constraints

- You must **name the root cause** for each suspect from this menu (each is used at least once): **non-sargable predicate**, **low selectivity (correct seq scan)**, **collation/opclass** (prefix `LIKE`), **stale statistics**.
- For each fixable case, apply the *minimal* fix — an expression index, a `text_pattern_ops` index, an `ANALYZE`, or "leave it alone, the planner is right."
- Don't just add indexes blindly. One of the four should end with "no index needed — the `Seq Scan` is correct."

## Hints

<details>
<summary>Suspect 1</summary>
The index stores `email`, but the query asks about `lower(email)`. Is that predicate sargable? What kind of index would make it so?
</details>

<details>
<summary>Suspect 2</summary>
How many distinct values does `status` have? What fraction of the table is `paid`? Check `pg_stats.n_distinct` and `most_common_freqs`. Is the planner wrong, or right?
</details>

<details>
<summary>Suspect 3</summary>
A prefix `LIKE 'user42%'` *is* a range and can use a B-tree — but only under one condition about the index's operator class / collation. Which one? (Lecture 1, §5.)
</details>

<details>
<summary>Suspect 4</summary>
Compare the *estimated* rows vs *actual* rows in the before-ANALYZE plan. A wildly wrong estimate on freshly loaded data points at one culprit.
</details>

## How success is judged

| Criterion | What "great" looks like |
|-----------|-------------------------|
| Correct diagnosis | Each suspect's root cause named correctly, backed by a plan or `pg_stats` output |
| Minimal fix | The smallest change that fixes it — or a justified "leave it" |
| Evidence | Before/after plans for the cases you fixed |
| Judgment | You correctly identify the one case where the `Seq Scan` should stay |

## Deliverable

`challenge-02.md` with, per suspect: the plan, the named root cause, the fix (or "no fix, and why"), and the after-plan where applicable.

## Submission

Commit to your portfolio under `c33-week-06/challenge-02/`.
