# Exercise 2 — Outer Joins & NULL Handling

**Goal:** Keep the rows an inner join would drop, and handle the `NULL`s outer joins manufacture — in the `SELECT` list, in `WHERE`, and with `COALESCE`. You'll also fall into (and climb out of) the classic "`WHERE` demotes my `LEFT JOIN`" trap.

**Estimated time:** 45 minutes.

## Setup

`crunch_shop` loaded (see [exercises/README.md](./README.md)). Recall the gaps you'll exploit:

- Customer 7 (Gizmo Inc) — no orders.
- Customer 6 (Foxtrot) — `region_id` is `NULL`.
- Product 6 (Gizmo-Pro) — `category_id` is `NULL`.
- Region 4 (West) — no customers.

> **SQLite:** tasks 4 and 8 use `RIGHT`/`FULL` joins — you need SQLite **3.39+**. Check `sqlite3 --version`. On Postgres 16 everything runs as-is.

## The eight tasks

1. **All customers, with their orders where they exist.** `customers LEFT JOIN orders`. Return customer `name`, `order_id`, `status`. *(Expect 8 rows — the 7 orders plus Gizmo Inc with `NULL`s.)*

2. **Customers who never ordered** — the anti-join. Take task 1 and add `WHERE o.order_id IS NULL`. *(Expect 1 row: Gizmo Inc.)*

3. **All products with their category name, keeping uncategorized products.** `products LEFT JOIN categories`. Return product `name`, category `name AS category`. Replace the `NULL` category with the text `'(uncategorized)'` using `COALESCE(cat.name, '(uncategorized)')`. *(Expect 6 rows; Gizmo-Pro shows `(uncategorized)`.)*

4. **All regions, with their customers where any.** Use a `RIGHT JOIN` so `regions` is preserved (or a `LEFT JOIN` with the tables swapped — your choice, note which you used). Return customer `name`, region `name`. *(Expect 7 rows — 6 region-having customers plus West with a NULL customer. Foxtrot's NULL region does NOT appear via this join; explain in a comment why.)*

5. **The `WHERE`-trap, on purpose.** Write this and record the row count:

   ```sql
   SELECT c.name, o.status
   FROM customers c
   LEFT JOIN orders o ON o.customer_id = c.customer_id
   WHERE o.status = 'shipped';
   ```

   Now explain in a comment: what happened to Gizmo Inc, and why did the `LEFT JOIN` behave like an `INNER JOIN`?

6. **Fix task 5 two ways.**
   - **(a)** Move the status test into the `ON` clause so Gizmo Inc survives with `NULL`.
   - **(b)** Keep the `WHERE` but add `OR o.status IS NULL`.
   Record both row counts and note how they differ.

7. **`COALESCE` for a friendly report.** All customers with their region name; show `'— none —'` where the region is missing. `customers LEFT JOIN regions`, `COALESCE(r.name, '— none —')`. *(Foxtrot should show `— none —`.)*

8. **Full reconciliation.** `customers FULL OUTER JOIN regions ON r.region_id = c.region_id`. Return customer `name`, region `name`, sorted so unmatched rows are visible. Identify: which customer has no region, and which region has no customer? *(Foxtrot: no region. West: no customer.)*

## Expected results (check yourself)

<details>
<summary>Reveal after attempting</summary>

- Task 1: **8** rows. Task 2: **1** row (Gizmo Inc).
- Task 3: **6** rows; Gizmo-Pro → `(uncategorized)`.
- Task 4: **7** rows. Foxtrot is absent because its `region_id` is `NULL`, and `NULL = anything` is never true, so it matches no region on the join key — a `RIGHT JOIN` preserving *regions* can't rescue a *customer* with a null key.
- Task 5: **4** rows — only the shipped orders (101, 102, 104, 106) survive; Gizmo Inc vanished. The `LEFT JOIN` produced its `NULL` row, then `WHERE o.status = 'shipped'` tested `NULL = 'shipped'` → unknown → dropped. The join was silently demoted to inner.
- Task 6(a): **8** rows — every customer is preserved; only *shipped* orders match, so Acme has 2 rows and Bolt Co, Dyeworks, Foxtrot, and Gizmo Inc each get one row with a NULL status. 6(b): **5** rows — the 4 shipped-order rows plus Gizmo Inc's one genuine NULL. Same predicate, very different result set: (a) keeps all customers, (b) keeps only shipped orders + order-less customers. Compare the actual rows, not just the counts.
- Task 8: Foxtrot has a NULL region; West has a NULL customer.

</details>

## Done when…

- [ ] `solutions.sql` has all eight queries with row-count comments.
- [ ] You wrote a one-sentence explanation for tasks 4 and 5.
- [ ] You used `COALESCE` correctly in tasks 3 and 7.
- [ ] You can state the rule: "a `WHERE` on a right-table column turns a `LEFT JOIN` into an `INNER JOIN` unless you spare the `NULL`s."

## Stretch

- Rewrite task 8's `FULL OUTER JOIN` using the `LEFT JOIN … UNION … IS NULL` recipe from Lecture 3, and confirm identical rows.
- Which employees have no assigned region? Write it as a `LEFT JOIN employees→regions … IS NULL`. *(Finn.)*

## Submission

Commit `solutions.sql` to your portfolio under `c33-week-02/exercise-02/`.
