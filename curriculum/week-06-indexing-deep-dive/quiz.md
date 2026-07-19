# Week 6 — Quiz

Fourteen questions. Lectures closed. Aim for 12/14 before Week 7.

---

**Q1.** A B-tree index over 200 million integer rows typically has a height of about:

- A) 200 levels.
- B) log₂(200M) ≈ 28 levels.
- C) 3–4 levels.
- D) It depends on the number of NULLs.

---

**Q2.** One B-tree on `orders(created_at)` can serve all of the following EXCEPT:

- A) `WHERE created_at BETWEEN a AND b`
- B) `ORDER BY created_at DESC`
- C) `SELECT max(created_at)`
- D) `WHERE date_part('hour', created_at) = 9`

---

**Q3.** `WHERE status = 'paid'` has a B-tree index but the planner chooses a `Seq Scan`. The most likely reason is:

- A) The index is corrupt.
- B) `status = 'paid'` is not selective enough — it keeps too large a fraction of the table.
- C) B-trees can't index text.
- D) You forgot to `COMMIT`.

---

**Q4.** Which predicate is **sargable**?

- A) `WHERE lower(email) = 'a@b.com'`
- B) `WHERE total_cents / 100 > 500`
- C) `WHERE created_at >= '2023-06-01' AND created_at < '2023-06-02'`
- D) `WHERE date(created_at) = '2023-06-01'`

---

**Q5.** `random_page_cost` is lowered from 4.0 to 1.1. The planner will:

- A) Refuse to use any index.
- B) Be **more** willing to choose index scans over sequential scans.
- C) Be less willing to use indexes.
- D) Ignore statistics entirely.

---

**Q6.** A `Bitmap Heap Scan` is chosen when:

- A) The predicate is extremely selective (one row).
- B) The predicate matches most of the table.
- C) A medium number of rows match — enough that reading heap pages in physical order beats scattered random fetches.
- D) The column is JSONB.

---

**Q7.** To accelerate `payload @> '{"plan":"pro"}'` on a JSONB column, use:

- A) A B-tree on `payload`.
- B) A hash index on `payload`.
- C) A GIN index on `payload`.
- D) A BRIN index on `payload`.

---

**Q8.** `jsonb_path_ops` (vs the default `jsonb_ops`) gives you:

- A) A larger index that supports every JSON operator.
- B) A smaller, faster index for `@>` containment, at the cost of key-existence (`?`) support.
- C) Full-text search.
- D) Nothing — it's an alias.

---

**Q9.** BRIN is a good fit when:

- A) The column has millions of distinct random values.
- B) You need exact single-row lookups.
- C) The column's values are physically correlated with row order (e.g. an append-only timestamp) and the table is huge.
- D) You need nearest-neighbor search.

---

**Q10.** Which index type can answer a nearest-neighbor query (`ORDER BY location <-> point LIMIT 10`)?

- A) B-tree
- B) Hash
- C) GiST
- D) BRIN

---

**Q11.** Given a composite index on `(customer_id, created_at)`, which query can it serve as a leftmost prefix?

- A) `WHERE created_at > '2023-01-01'`
- B) `WHERE customer_id = 42`
- C) `WHERE total_cents > 500`
- D) `WHERE status = 'paid'`

---

**Q12.** In a composite index, you should generally place:

- A) Range-filtered columns before equality-filtered columns.
- B) Equality-filtered columns before range-filtered columns.
- C) The largest column first.
- D) Columns in alphabetical order.

---

**Q13.** `INCLUDE (total_cents)` on an index does what?

- A) Makes `total_cents` part of the sorted key.
- B) Adds `total_cents` as payload in the leaf so the index can satisfy a query without a heap fetch — enabling an index-only scan.
- C) Forces a unique constraint on `total_cents`.
- D) Excludes `total_cents` from the index.

---

**Q14.** An `Index Only Scan` reports `Heap Fetches: 91240`. The fix is:

- A) Drop the index.
- B) Add more `INCLUDE` columns.
- C) `VACUUM` the table to refresh the visibility map.
- D) Increase `work_mem`.

---

## Answer key

<details>
<summary>Reveal after attempting</summary>

1. **C** — B-trees are wide and shallow; 3–4 levels index enormous tables.
2. **D** — wrapping the column in `date_part(...)` is non-sargable; the index stores raw `created_at`, not the extracted hour. A, B, C all reduce to positions in sorted order.
3. **B** — low selectivity. With ~5 distinct statuses, `paid` keeps ~20% of rows; a seq scan is cheaper than millions of random heap fetches. The planner is correct.
4. **C** — the column is bare on one side and compared to constants. A, B, D all transform the column, defeating the index.
5. **B** — lowering `random_page_cost` makes index (random) access look cheaper, so the planner favors index scans. Common SSD tuning.
6. **C** — medium selectivity; the bitmap re-sorts fetches into physical page order.
7. **C** — GIN is the inverted index for JSONB containment.
8. **B** — `jsonb_path_ops` is smaller/faster for `@>` but drops `?`/`?|`/`?&` support.
9. **C** — BRIN relies on physical/value correlation; ideal for append-only time-series on huge tables.
10. **C** — GiST supports the distance operator `<->` for KNN. B-trees cannot.
11. **B** — the first column of the index is a valid prefix. A and D skip the leading column; C isn't in the index.
12. **B** — equality before range: once you range-scan a column, later columns can't be used efficiently.
13. **B** — `INCLUDE` carries non-key payload in the leaf, enabling index-only scans without imposing sort order.
14. **C** — high heap fetches mean a stale visibility map; `VACUUM` refreshes it so the scan can stay index-only.

</details>

**Scoring:** 12+ → move to Week 7. 9–11 → re-read the missed lecture sections. <9 → re-read all three lectures and redo Exercises 1 and 3.
