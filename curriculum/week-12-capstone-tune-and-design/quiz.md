# Week 12 — Quiz

Twelve questions. Lectures closed. Aim 10/12 before you call C33 done. Mix of multiple-choice and short-answer.

---

**Q1.** The five steps of the tuning loop, in order, are:

- A) Read the plan → measure → verify → change → repeat
- B) Measure → read the plan → change one thing → verify → repeat
- C) Change → measure → read the plan → repeat → verify
- D) Index → benchmark → read the plan → measure → repeat

---

**Q2.** When hunting for the query to optimize first, you should sort `pg_stat_statements` by:

- A) `mean_exec_time` — the slowest single query
- B) `calls` — the most frequent query
- C) `total_exec_time` — where time actually accumulates
- D) `rows` — the query returning the most data

---

**Q3.** In an `EXPLAIN ANALYZE` plan, a large `Rows Removed by Filter` under a `Seq Scan` most directly suggests:

- A) The sort spilled to disk
- B) A selective predicate with no supporting index
- C) The join algorithm is wrong
- D) Statistics are stale

---

**Q4.** The planner estimates `rows=40` but the node shows `actual rows=90000`. Your **first** move is:

- A) Add an index
- B) Rewrite the query
- C) Run `ANALYZE` / raise the column's statistics target
- D) Increase `work_mem`

---

**Q5.** Why report **p95** latency instead of the fastest single run?

*(Short answer, one or two sentences.)*

---

**Q6.** Which is a genuine cost of adding an index? (Choose all that apply.)

- A) Slower `INSERT`/`UPDATE`/`DELETE`
- B) Disk and cache consumption
- C) Slower `SELECT` on queries that use it
- D) More planning options for the optimizer to weigh

---

**Q7.** EAV (entity–attribute–value) primarily trades away:

- A) Write throughput for read speed
- B) Type safety, constraints, and query simplicity for schema flexibility
- C) Disk space for query speed
- D) Nothing — it is always the right choice for user data

---

**Q8.** An N+1 access pattern appears in `pg_stat_statements` as a query with:

- A) High mean time, one call
- B) High total time, high call count, low mean time
- C) Low total time, low call count
- D) A `Seq Scan` in its plan

---

**Q9.** Money should be stored as `numeric`, never `float`, because:

*(Short answer.)*

---

**Q10.** `Buffers: shared read=38891` in a plan means roughly:

- A) 38,891 rows were returned
- B) ~304 MB was read from disk (38891 × 8 KB pages)
- C) 38,891 index entries were scanned
- D) The query used 38,891 KB of `work_mem`

---

**Q11.** You have indexes on `(customer_id)` and `(customer_id, created_at)`. The first is likely:

- A) Essential — keep both always
- B) Redundant for `customer_id`-only lookups, since it is a prefix of the composite
- C) Faster than the composite for every query
- D) Required for the foreign key to work

---

**Q12.** Name **three** conditions under which a senior engineer decides to *stop* tuning a query that could still be made faster.

*(Short answer.)*

---

## Answer key

<details>
<summary>Reveal after attempting</summary>

1. **B** — Measure → read the plan → change one thing → verify → repeat.
2. **C** — `total_exec_time`. A fast query run millions of times can dominate; optimize where time lives.
3. **B** — a selective `WHERE` scanning the whole table and discarding most rows wants an index.
4. **C** — a 1000×+ estimate error is a statistics problem first. Fix the stats before changing the query or adding an index — the planner may then pick a good plan on its own.
5. **p95** captures the tail users actually feel; a single fast run reflects a warm cache and hides variance. A budget is about the distribution, not the best case.
6. **A, B, D** — indexes tax writes, consume disk/cache, and add planner options. They do **not** slow the `SELECT`s that use them (C is false).
7. **B** — EAV buys schema-less flexibility and pays with lost types, lost constraints, and painful multi-join/pivot queries.
8. **B** — many fast calls: high total, high count, low mean. The fix is one set-based query.
9. Floating-point cannot represent decimal fractions like 0.10 exactly, so sums and comparisons accumulate rounding errors — unacceptable for accounting. `numeric` is exact.
10. **B** — pages are 8 KB; 38891 × 8 KB ≈ 304 MB read from disk.
11. **B** — `(customer_id)` is a prefix of `(customer_id, created_at)`, so the composite already serves `customer_id`-only lookups. Keep it only if a `customer_id`-alone index is meaningfully smaller/hotter; otherwise drop it. (Note: an FK does not *require* an index, though indexing FK columns is usually wise.)
12. Any three of: the query meets its stated latency budget (at p95 **and** p99); it has headroom for projected data growth; it is no longer a meaningful share of `total_exec_time`; the next fix would cost more (writes, disk, complexity, maintenance) than it saves.

</details>

If you scored 10+: you have the week. 7–9: re-read the lecture behind each miss. <7: re-read all three lectures before the capstone — the report demands this fluency.
