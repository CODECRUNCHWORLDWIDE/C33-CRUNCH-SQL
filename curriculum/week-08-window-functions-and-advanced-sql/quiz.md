# Week 8 — Quiz

Twelve self-check questions. Lectures closed. Aim for 10/12 before Week 9. A mix of multiple-choice and short-answer.

---

**Q1.** What is the defining difference between a window function and `GROUP BY`?

- A) Window functions are faster.
- B) Window functions return one row per input row; `GROUP BY` collapses rows into groups.
- C) `GROUP BY` can't compute sums.
- D) Window functions can't be partitioned.

---

**Q2.** Given the ordered values `100, 90, 90, 80` in one partition, what does `RANK()` produce?

- A) `1, 2, 3, 4`
- B) `1, 2, 2, 3`
- C) `1, 2, 2, 4`
- D) `1, 1, 2, 3`

---

**Q3.** …and what does `DENSE_RANK()` produce for the same input?

- A) `1, 2, 2, 3`
- B) `1, 2, 2, 4`
- C) `1, 2, 3, 4`
- D) `1, 1, 3, 4`

---

**Q4.** Why does `SELECT * FROM t WHERE ROW_NUMBER() OVER (ORDER BY x) = 1` fail?

- A) `ROW_NUMBER` needs a `PARTITION BY`.
- B) Window functions are evaluated after `WHERE`, so you can't filter on them there.
- C) `ROW_NUMBER` isn't a real function.
- D) `WHERE` doesn't allow `ORDER BY`.

---

**Q5.** You add `ORDER BY d` to `SUM(amount) OVER (PARTITION BY cat …)` and your totals suddenly climb row by row. Why?

- A) `ORDER BY` sorts the amounts.
- B) With `ORDER BY` and no explicit frame, the default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — a running total.
- C) `PARTITION BY` stopped working.
- D) It's a bug in the engine.

---

**Q6.** Which frame clause gives a **7-row trailing** moving average (current row + 6 before)?

- A) `ROWS BETWEEN 7 PRECEDING AND CURRENT ROW`
- B) `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`
- C) `RANGE BETWEEN 7 PRECEDING AND 7 FOLLOWING`
- D) `ROWS BETWEEN CURRENT ROW AND 6 FOLLOWING`

---

**Q7.** `LAST_VALUE(x) OVER (ORDER BY d)` surprisingly returns the *current* row's value instead of the partition's last. Why, and what's the fix? *(short answer)*

---

**Q8.** In the gaps-and-islands trick, why is `value − ROW_NUMBER()` constant across a run of consecutive integers? *(short answer)*

---

**Q9.** What are the three steps of sessionizing an event stream, in order?

- A) `GROUP BY` user, sort, count.
- B) `LAG` to measure the gap, `CASE` to flag a new session, running `SUM` to number sessions.
- C) `RANK`, `NTILE`, `LEAD`.
- D) Join, pivot, aggregate.

---

**Q10.** Which counts distinct peer-groups rather than physical rows or value ranges?

- A) `ROWS`
- B) `RANGE`
- C) `GROUPS`
- D) `PARTITION`

---

**Q11.** You need "total sales per category, one row per category, for a report." Window function or `GROUP BY`? Why in one clause? *(short answer)*

---

**Q12.** What does the third argument to `LAG(expr, offset, default)` do?

- A) Sets the partition.
- B) Provides a value to return when there is no row at that offset (e.g. the first row).
- C) Rounds the result.
- D) Reverses the order.

---

## Answer key

<details>
<summary>Reveal after attempting</summary>

1. **B** — window functions keep every input row; `GROUP BY` collapses.
2. **C** — `1, 2, 2, 4`. `RANK` shares the tied rank and then *skips* (gap).
3. **A** — `1, 2, 2, 3`. `DENSE_RANK` shares the rank but does *not* skip.
4. **B** — window functions run after `WHERE`/`GROUP BY`/`HAVING` and before `ORDER BY`/`LIMIT`. Wrap in a CTE and filter in the outer query.
5. **B** — adding `ORDER BY` without an explicit frame switches to the default `RANGE … UNBOUNDED PRECEDING AND CURRENT ROW`, i.e. a running total. Set the frame explicitly if you wanted the full partition total.
6. **B** — `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` = 6 prior + current = 7 rows.
7. The default frame ends at `CURRENT ROW`, so "the last row of the frame" *is* the current row. Fix: extend the frame with `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` (or use `FIRST_VALUE` over a `DESC` order).
8. Consecutive integers increase by 1 each step; `ROW_NUMBER()` also increases by 1 each step within the ordered partition. Their difference is therefore constant along a run, and jumps only where a gap breaks the "+1 each step" pattern — making the difference a natural group key per island.
9. **B** — `LAG` measures the inactivity gap, a `CASE` flags rows whose gap exceeds the threshold (plus the first event) as `1`, and a running `SUM` of that flag assigns an increasing session number.
10. **C** — `GROUPS` counts distinct peer-groups (rows sharing an `ORDER BY` value). `ROWS` counts physical rows; `RANGE` counts by value.
11. **`GROUP BY`** — you want exactly one output row per category (a summary), and you're discarding the detail rows. A window function would repeat the total on every detail row, which is the wrong shape for a report line.
12. **B** — it's the default returned when no row exists at the requested offset (e.g. `LAG(x, 1, 0)` yields `0` for the first row instead of `NULL`).

</details>

**Scoring:** 11–12 → move to the mini-project and Week 9. 8–10 → re-read the lecture behind each miss. <8 → re-read all three lectures from the top; window functions reward a second pass.
