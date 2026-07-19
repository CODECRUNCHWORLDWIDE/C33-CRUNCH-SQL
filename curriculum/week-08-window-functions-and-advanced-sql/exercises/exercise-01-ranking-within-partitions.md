# Exercise 1 — Ranking Within Partitions

**Goal:** Rank rows *inside* groups and extract the top N per group — the single most common window-function task in a real job.

**Estimated time:** 90 minutes.

## Setup

A small product catalog with sales per product, grouped by category. Works in PostgreSQL and SQLite unchanged.

```sql
CREATE TABLE products (
  id        INTEGER PRIMARY KEY,
  name      TEXT,
  category  TEXT,
  units     INTEGER
);

INSERT INTO products (id, name, category, units) VALUES
  (1,  'Basil',      'herbs',    120),
  (2,  'Thyme',      'herbs',     95),
  (3,  'Mint',       'herbs',     95),
  (4,  'Rosemary',   'herbs',     60),
  (5,  'Carrot',     'veg',      300),
  (6,  'Potato',     'veg',      300),
  (7,  'Onion',      'veg',      250),
  (8,  'Leek',       'veg',       40),
  (9,  'Apple',      'fruit',    500),
  (10, 'Pear',       'fruit',    220),
  (11, 'Plum',       'fruit',    220),
  (12, 'Fig',        'fruit',     90);
```

## Tasks

Write one query per task. Save each into `exercise-01.sql`.

### 1. Rank every product within its category by units sold (highest first)

Show `name`, `category`, `units`, and three columns side by side: `ROW_NUMBER`, `RANK`, `DENSE_RANK`. Order the output by category, then units descending.

**Expected result** (note how the three ranks differ on the ties — Mint/Thyme at 95, Potato/Carrot at 300, Pear/Plum at 220):

```
   name   | category | units | row_num | rnk | dense_rnk
----------+----------+-------+---------+-----+----------
 Apple    | fruit    |  500  |    1    |  1  |    1
 Pear     | fruit    |  220  |    2    |  2  |    2
 Plum     | fruit    |  220  |    3    |  2  |    2
 Fig      | fruit    |   90  |    4    |  4  |    3
 Basil    | herbs    |  120  |    1    |  1  |    1
 Thyme    | herbs    |   95  |    2    |  2  |    2
 Mint     | herbs    |   95  |    3    |  2  |    2
 Rosemary | herbs    |   60  |    4    |  4  |    3
 Carrot   | veg      |  300  |    1    |  1  |    1
 Potato   | veg      |  300  |    2    |  1  |    1
 Onion    | veg      |  250  |    3    |  3  |    2
 Leek     | veg      |   40  |    4    |  4  |    3
```

(Which of Pear/Plum, Mint/Thyme, Carrot/Potato gets `row_num` 2 vs 3 is arbitrary unless you add a tiebreaker — see task 4.)

### 2. Return only the top 2 products per category

Use `ROW_NUMBER` in a CTE, then filter `WHERE rn <= 2` in the outer query. Remember: you **cannot** filter on a window function directly in `WHERE`.

### 3. Same top-2, but include ties

Swap `ROW_NUMBER` for `RANK` (or `DENSE_RANK`) so that when two products tie for 2nd, **both** appear. Compare the row counts of tasks 2 and 3 — they differ in `fruit` and `veg`.

### 4. Make it deterministic

Task 1's `row_num` is non-reproducible because tied rows have no defined order. Add a tiebreaker so the output is stable across runs. (Hint: add `, id` to the window's `ORDER BY`.)

### 5. Bucket products into 3 tiers by units, across the whole catalog

Use `NTILE(3) OVER (ORDER BY units DESC)` — no partition — to split all 12 products into three tiers of four. Which products land in tier 1?

## Done when…

- [ ] `exercise-01.sql` contains all five queries.
- [ ] Task 1's output matches the expected table (ranks correct on every tie).
- [ ] You can state, in one sentence each, why `RANK` shows `1,1,3` for the `veg` category while `DENSE_RANK` shows `1,1,2`.
- [ ] Task 2 returns 6 rows; task 3 returns 8 (ties in fruit and veg add two).
- [ ] Task 4's output is identical on two consecutive runs.

## Stretch

- Add a column showing each product's **percent of its category's total units**: `100.0 * units / SUM(units) OVER (PARTITION BY category)`.
- Add `PERCENT_RANK()` per category and explain what a value of `0` means (it's the top row).
- Return, for each category, *only* the single best-selling product using `DISTINCT ON` (PostgreSQL) or a `ROW_NUMBER = 1` filter (both engines). Which reads more clearly?

## Hints

<details>
<summary>Top-N-per-group skeleton</summary>

```sql
WITH ranked AS (
  SELECT name, category, units,
         ROW_NUMBER() OVER (PARTITION BY category ORDER BY units DESC, id) AS rn
  FROM products
)
SELECT * FROM ranked WHERE rn <= 2 ORDER BY category, rn;
```

Change `ROW_NUMBER` → `RANK` for task 3.
</details>

## Submission

Commit `exercise-01.sql` under `c33-week-08/exercises/`.
