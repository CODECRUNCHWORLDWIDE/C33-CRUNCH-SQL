# Lecture 2 — Scans, Joins, and the Cost Model

> **Duration:** ~2 hours. **Outcome:** You can name every scan type and join algorithm PostgreSQL uses, predict which one it will pick for a given query, and explain the cost formula and statistics that drive the decision.

Lecture 1 taught you to *read* a plan. This lecture teaches you to *predict* one. The planner's job is to choose, for each table, **how to read it** (a scan), and for each pair of tables, **how to combine them** (a join). There is a small, fixed menu of options for each, and the planner picks by estimating cost. Once you know the menu and the cost math, plans stop being mysterious — you can look at a query and its data and say, before running it, "this will be a hash join over two sequential scans," and be right.

## 1. The scan menu — five ways to read a table

Every table access at the leaf of a plan is one of these:

| Scan type | What it does | Best when |
|-----------|--------------|-----------|
| **Seq Scan** | Read every page of the table top to bottom | You want most rows, or the table is small, or no useful index exists |
| **Index Scan** | Walk a B-tree to matching keys, then fetch each row from the heap | Highly selective predicate (returns few rows) |
| **Index Only Scan** | Walk the index and return values *without touching the heap* | The index covers every column the query needs |
| **Bitmap Index Scan** + **Bitmap Heap Scan** | Build a bitmap of matching page locations, then read those pages in physical order | Medium selectivity; too many rows for a plain index scan, too few for a seq scan |

### Seq Scan is not the enemy

Beginners see `Seq Scan` and panic. Stop. A sequential scan reads pages in physical order — the fastest possible disk access pattern — and if you are returning half the table, it is *the correct choice*. An index scan that fetches 500,000 rows one at a time, each a random page read, is far slower than one sequential sweep. The planner knows this. `Seq Scan` is only a problem when the query is **selective** (returns a small fraction) and an index *should* have been used but wasn't.

### Index Scan vs. Bitmap: selectivity decides

For a selective predicate, PostgreSQL walks the index and, for each match, does a random fetch into the heap. Random fetches are expensive, so this only wins when there are few matches. As the number of matches grows, a plain Index Scan degrades — you'd do thousands of random page reads, possibly hitting the same page many times.

The **bitmap** path fixes this. A Bitmap Index Scan reads the index and builds a bitmap of *which heap pages* contain matches. Then a Bitmap Heap Scan reads those pages **once each, in physical order.** You'll see it in plans like:

```
Bitmap Heap Scan on orders  (cost=... rows=4200 ...)
  Recheck Cond: (status = 'shipped')
  ->  Bitmap Index Scan on orders_status_idx  (cost=... rows=4200 ...)
        Index Cond: (status = 'shipped')
```

The `Recheck Cond` appears because if the bitmap grows too large it becomes "lossy" (tracks pages, not exact rows) and the condition must be re-verified per row. Bitmap scans also let PostgreSQL **combine multiple indexes** with `BitmapAnd`/`BitmapOr` — one index per condition, bitmaps merged.

### Index Only Scan — the fast path

If every column your query references is *in the index*, PostgreSQL can answer from the index alone and never touch the table heap:

```sql
CREATE INDEX ON orders (customer_id) INCLUDE (total);
EXPLAIN (ANALYZE) SELECT total FROM orders WHERE customer_id = 1234;
```

```
Index Only Scan using orders_customer_id_total_idx on orders
  Index Cond: (customer_id = 1234)
  Heap Fetches: 0
```

`Heap Fetches: 0` is the win — zero heap access. But watch that number: PostgreSQL's MVCC means the index doesn't know if a row is *visible* to your transaction, so it consults the **visibility map**. On a table with many recent writes, `Heap Fetches` can be high, and the "index only" scan quietly becomes a normal one. Keep tables `VACUUM`ed to keep the visibility map fresh (Week 5/9 material).

## 2. The join menu — three algorithms

When two tables (or a table and an intermediate result) must be combined, the planner chooses one of exactly three algorithms.

| Join | How it works | Cost shape | Wins when |
|------|--------------|-----------|-----------|
| **Nested Loop** | For each row of the outer input, scan the inner input for matches | O(outer × inner), unless inner is indexed | Outer side is tiny, **or** inner side has an index on the join key |
| **Hash Join** | Build a hash table on the smaller input, probe it with the larger | O(outer + inner), needs memory for the hash | Large, unsorted inputs joined on equality |
| **Merge Join** | Sort both inputs on the join key, then walk them in lockstep | O(n log n) to sort + O(n) to merge | Inputs already sorted (e.g., from an index), or very large |

### Nested Loop

The simplest join. It shines in two opposite cases: when the outer input is only a handful of rows (so "for each outer row" isn't many iterations), or when the inner input has an index on the join column (so each inner lookup is cheap):

```
Nested Loop
  ->  Index Scan on customers  (rows=1)         -- one customer
  ->  Index Scan on orders     (rows=8 loops=1) -- that customer's orders, via index
```

The Nested Loop is also the villain of the mis-estimated plan (Lecture 1): if the planner thinks the outer side is 1 row but it's actually 50,000, the inner side runs 50,000 times and the query explodes. **A Nested Loop with a large `loops` count on the inner side is the number-one performance smell.**

### Hash Join

For two big tables joined on `=`, hashing wins. PostgreSQL builds a hash table from the smaller input (the `Hash` node you saw in Lecture 1), then streams the larger input through it:

```
Hash Join  (cost=... rows=... )
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o
  ->  Hash
        ->  Seq Scan on customers c
```

The catch: the hash table must fit in `work_mem`. If it doesn't, PostgreSQL splits the join into **batches** spilled to disk. You'll see `Batches: 4` (more than 1) and `Disk Usage` — a sign to raise `work_mem` (Lecture 3). Hash joins only work for **equality** conditions.

### Merge Join

If both inputs arrive sorted on the join key — most naturally from index scans — the planner can zip them together in one pass, like merging two sorted lists. No hash table, low memory, and it handles range conditions and huge inputs gracefully. The cost is the sort, so merge join wins when the sort is free (an index already provides the order) or when inputs are so large that hashing would spill badly.

## 3. The cost model — where the numbers come from

The planner turns each candidate plan into a single number by summing two things: **estimated I/O** (pages read) and **estimated CPU** (rows and expressions processed). Each is weighted by a cost constant:

| Parameter | Default | Meaning |
|-----------|--------:|---------|
| `seq_page_cost` | `1.0` | Cost to read one page sequentially (the reference unit) |
| `random_page_cost` | `4.0` | Cost to read one page at a random location |
| `cpu_tuple_cost` | `0.01` | Cost to process one row |
| `cpu_index_tuple_cost` | `0.005` | Cost to process one index entry |
| `cpu_operator_cost` | `0.0025` | Cost to evaluate one operator/function |
| `parallel_setup_cost` | `1000` | Fixed cost to start parallel workers |

A rough sequential scan cost is:

```
cost ≈ (pages × seq_page_cost) + (rows × cpu_tuple_cost) + (rows × cpu_operator_cost × conditions)
```

So a 10,000-page table with 1,000,000 rows and one filter costs about `10000×1.0 + 1000000×0.01 + 1000000×0.0025 ≈ 10000 + 10000 + 2500 = 22500`. That's the `cost` you see in the plan.

### Why `random_page_cost = 4`

The default says a random page read costs 4× a sequential one — a model of spinning disks, where the head must seek. **On SSDs and cloud block storage, random reads are far closer to sequential.** Lowering `random_page_cost` to `1.1`–`2.0` on SSD-backed systems tells the planner index scans are cheaper than it assumes, and often flips it toward using your indexes. This is one of the highest-value single settings on modern hardware:

```sql
-- Session-level test; set in postgresql.conf to make permanent
SET random_page_cost = 1.1;
```

Change it, re-`EXPLAIN`, and watch Seq Scans turn into Index Scans. (Only make it permanent after measuring — see Lecture 3.)

## 4. Statistics — the planner's model of your data

Cost formulas need one input above all: **how many rows will match?** The planner answers that from statistics gathered by the `ANALYZE` command (or by autovacuum's auto-analyze). These live in the system catalog and are surfaced through the `pg_stats` view.

```sql
ANALYZE orders;   -- collect fresh stats for this table

SELECT attname, n_distinct, most_common_vals, most_common_freqs, null_frac
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

Key columns of `pg_stats`:

| Column | What it tells the planner |
|--------|---------------------------|
| `n_distinct` | Number of distinct values (or negative = ratio of distinct to total) |
| `most_common_vals` (MCV) | The most frequent values, listed explicitly |
| `most_common_freqs` | How frequent each MCV is |
| `histogram_bounds` | Buckets describing the spread of the remaining values |
| `null_frac` | Fraction of rows that are NULL |
| `correlation` | How closely physical row order matches this column's order (−1..1) |

From these, the planner estimates selectivity. For `WHERE status = 'shipped'`, it checks the MCV list: if `'shipped'` is there with frequency 0.42, it estimates `0.42 × total_rows`. For a range like `WHERE total BETWEEN 100 AND 200`, it uses the histogram. The `correlation` column drives whether an index scan will be sequential-ish or random — a high correlation makes index scans much cheaper (this is what a BRIN index exploits, from Week 6).

### The statistics target

How detailed are these stats? Controlled by `default_statistics_target` (default `100` = up to 100 MCVs and 100 histogram buckets per column). Higher = more accurate estimates, slower `ANALYZE`, bigger catalog. You can raise it globally or per-column:

```sql
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;
```

Raise it for columns with skewed or many distinct values where estimates are going wrong. Leave the default for everything else.

## 5. Putting it together — predicting a plan

Given `SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id WHERE c.country = 'US'`, reason like the planner:

1. **How selective is `country = 'US'`?** Check `pg_stats`. If 'US' is 30% of customers, that's a lot of rows — favor a Seq Scan on customers.
2. **How will orders be joined?** Both sides are large and it's an equality join → **Hash Join** is likely, building the hash on the smaller (customers) side.
3. **Would an index help?** Only if `country = 'US'` were *selective* (say 0.5% of customers) would an index scan on customers beat a seq scan.

Run it and confirm. The point of this lecture is that you should be able to *predict* the plan's shape and then verify — that loop is how plan-reading becomes intuition.

## 6. SQLite's cost model, briefly

SQLite's planner is simpler but the same in spirit. It runs its own `ANALYZE`, which populates the `sqlite_stat1` table with per-index row-count summaries. Without `ANALYZE`, SQLite uses crude defaults and can pick poorly on skewed data. SQLite considers essentially two access methods (full `SCAN` and indexed `SEARCH`) and prefers indexes that avoid a separate sort. It has no hash join — it joins by nested loops driven by indexes, which is why good indexes matter even more in SQLite. Run `ANALYZE;` after bulk-loading a SQLite database and re-check `EXPLAIN QUERY PLAN`.

## 7. Check yourself

- Name the five scan types and give the one situation where each is the right choice.
- Why is a Seq Scan sometimes *faster* than an Index Scan even when an index exists?
- What problem does a Bitmap Heap Scan solve that a plain Index Scan has? What is `Recheck Cond`?
- Name the three join algorithms and the query shape where each wins.
- A Nested Loop shows `loops=50000` on its inner Index Scan. Why is that a red flag, and what usually caused it?
- What does `random_page_cost = 4.0` assume about the hardware, and what should you set it to on SSD — and why?
- Which `pg_stats` column would you check to see whether `WHERE status = 'shipped'` is estimated well?

When these are comfortable, move to [Lecture 3 — Rewriting Queries and Tuning Knobs](./03-rewriting-queries-and-tuning-knobs.md).

## Further reading

- **PostgreSQL — Planner cost constants:** <https://www.postgresql.org/docs/current/runtime-config-query.html>
- **PostgreSQL — How the planner uses statistics:** <https://www.postgresql.org/docs/current/planner-stats.html>
- **PostgreSQL — Row estimation examples:** <https://www.postgresql.org/docs/current/row-estimation-examples.html>
- **SQLite — The query optimizer overview:** <https://www.sqlite.org/optoverview.html>
