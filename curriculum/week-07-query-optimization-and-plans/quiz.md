# Week 7 — Quiz

Thirteen questions. Lectures closed. Aim for 11/13 before Week 8. Mix of multiple-choice and short-answer.

---

**Q1.** Which command actually *executes* the query it is given?

- A) `EXPLAIN q`
- B) `EXPLAIN (ANALYZE) q`
- C) `EXPLAIN (COSTS) q`
- D) `EXPLAIN (VERBOSE) q`

---

**Q2.** In `cost=0.43..18.62`, what is `0.43`?

- A) The total cost of the node.
- B) The number of rows.
- C) The startup cost — cost before the first row is emitted.
- D) The time in milliseconds.

---

**Q3.** In a plan tree, data flows:

- A) From the top node down to the leaves.
- B) From the leaves (most indented) up to the top node.
- C) Left to right.
- D) In whatever order the planner logs it.

---

**Q4.** A node shows `rows=1` estimated but `actual rows=42000`. Why does this matter most?

- A) It doesn't; only time matters.
- B) The planner built its plan on a wrong row count, so its downstream choices (like Nested Loop) are likely wrong.
- C) It means the index is corrupt.
- D) It means `work_mem` is too low.

---

**Q5.** A node reads `actual time=0.3..0.8 rows=4 loops=25000`. Roughly how much total time did it consume?

- A) ~0.8 ms
- B) ~4 ms
- C) ~20,000 ms
- D) ~200 ms

---

**Q6.** Why might the planner correctly choose a Seq Scan even though a relevant index exists?

- A) Seq Scans are always faster.
- B) The query returns a large fraction of the table, so one sequential sweep beats many random index fetches.
- C) The index is disabled.
- D) `EXPLAIN` can't see indexes.

---

**Q7.** What problem does a **Bitmap Heap Scan** solve compared to a plain Index Scan?

*(Short answer, one sentence.)*

---

**Q8.** Match each join to its best case:

1. Nested Loop  2. Hash Join  3. Merge Join

- A) Two large unsorted inputs joined on equality
- B) Tiny outer input, or indexed inner side
- C) Both inputs already sorted on the join key

---

**Q9.** `random_page_cost` defaults to `4.0`. On SSD-backed storage you should usually:

- A) Raise it to 8.
- B) Leave it — it's hardware-independent.
- C) Lower it toward 1.1–2.0 so index scans are costed fairly.
- D) Set it to 0.

---

**Q10.** Which `pg_stats` column tells the planner the fraction of NULLs in a column?

- A) `n_distinct`
- B) `null_frac`
- C) `correlation`
- D) `histogram_bounds`

---

**Q11.** Why is `WHERE date_trunc('day', created_at) = '2026-07-16'` a performance problem, and how do you rewrite it?

*(Short answer.)*

---

**Q12.** `NOT IN (SELECT ... )` where the subquery can return NULL is dangerous because:

- A) It's slower to type.
- B) A single NULL makes the whole predicate never TRUE, silently returning zero rows; and it often plans poorly. Use `NOT EXISTS`.
- C) It locks the table.
- D) It requires a covering index.

---

**Q13.** You see `Sort Method: external merge  Disk: 84656kB` in a plan. What happened, and which knob addresses it?

*(Short answer.)*

---

## Answer key

<details>
<summary>Reveal after attempting</summary>

1. **B** — `EXPLAIN (ANALYZE)` really runs the query (dangerous on writes; wrap in `BEGIN … ROLLBACK`).
2. **C** — startup cost, the cost before the first row can be emitted. High for sorts, near-zero for seq scans.
3. **B** — leaves execute first and feed upward; the top node produces the final result.
4. **B** — a wrong row estimate corrupts every downstream decision; the classic cause of a Nested Loop that explodes.
5. **C** — per-loop time is multiplied by loops: `0.8 ms × 25000 ≈ 20,000 ms`. Always multiply by `loops`.
6. **B** — for low selectivity (many rows), one sequential sweep beats thousands of random heap fetches. Seq Scan is often correct.
7. **Q7:** It avoids doing many random single-row heap fetches by collecting all matching page locations into a bitmap and reading those pages once each, in physical order (good for medium selectivity; may add a `Recheck Cond`).
8. **1→B, 2→A, 3→C.**
9. **C** — lower it toward 1.1–2.0; the default 4.0 models spinning-disk seeks, which SSDs largely don't have.
10. **B** — `null_frac`.
11. **Q11:** Wrapping the indexed column in `date_trunc(...)` makes it non-sargable — the index on `created_at` can't be used. Rewrite as a bare-column range: `created_at >= '2026-07-16' AND created_at < '2026-07-17'`.
12. **B** — three-valued logic: any NULL in the list makes `NOT IN` never TRUE, silently returning zero rows; and it plans poorly. Prefer `NOT EXISTS` (NULL-safe, plans as an Anti Join).
13. **Q13:** The sort exceeded `work_mem` and spilled to temporary files on disk (much slower). Raise `work_mem` for that query/session (carefully — it's per-node, per-connection, so don't set it huge globally).

</details>

If you scored 11+: move on. 8–10: re-read the lecture behind each miss. <8: re-read Lectures 1 and 2 from the top before the mini-project.
