# Exercise 3 — Self-Joins & Set Operations

**Goal:** Join a table to itself (the org chart), then combine result sets with `UNION`, `INTERSECT`, and `EXCEPT`. You'll also see, first-hand, where a set operator is cleaner than a join and vice versa.

**Estimated time:** 45 minutes.

## Setup

`crunch_shop` loaded (see [exercises/README.md](./README.md)). Two facts you'll lean on:

- `employees.manager_id` points back at `employees.employee_id`. Ada (id 1) is the CEO — her `manager_id` is `NULL`.
- Customers' regions are `{North, South, East}`; suppliers' regions are `{North, South, East, West}`. West has a supplier but no customer.

## Part A — Self-joins

1. **Each employee beside their manager's name.** Self-join `employees e` to `employees m` on `m.employee_id = e.manager_id` (inner). Return `e.name AS employee`, `m.name AS manager`. *(Expect 5 rows — Ada is dropped by the inner join. Why?)*

2. **Keep the CEO.** Change task 1 to a `LEFT JOIN` so Ada appears with a `NULL` manager. Wrap the manager name in `COALESCE(m.name, '(none — top of chart)')`. *(Expect 6 rows.)*

3. **Same-region coworkers.** Find pairs of *different* employees who share a region. Self-join `employees a` to `employees b` on `a.region_id = b.region_id AND a.employee_id < b.employee_id`. Return both names and the region. *(Why the `<`? What would happen without it?)*

4. **Skip-level view.** Show each employee, their manager, and their manager's manager (grand-manager). Chain two self-joins: `employees e` → `employees m` (e's manager) → `employees g` (m's manager). Use `LEFT JOIN`s so people high in the chart aren't dropped. Return `e.name`, `m.name AS manager`, `g.name AS grand_manager`.

## Part B — Set operations

5. **All region IDs that appear as a customer's OR a supplier's region.** `UNION` the two `SELECT region_id` queries (filter out `NULL`s). *(Expect `{1, 2, 3, 4}`.)*

6. **Regions used by both a customer AND a supplier.** Swap `UNION` for `INTERSECT`. *(Expect `{1, 2, 3}`.)*

7. **Regions with a supplier but NO customer.** `suppliers' regions EXCEPT customers' regions`. *(Expect `{4}` — West.)* Then reverse the operands (`customers EXCEPT suppliers`) and record what you get and why.

8. **`UNION` vs `UNION ALL`.** Run task 5 once with `UNION` and once with `UNION ALL`. Record both row counts and explain the difference in a comment.

9. **Labeled combined feed.** Produce a single two-column list of all parties: a literal `'customer'`/`'supplier'` tag plus the name. `SELECT 'customer', name FROM customers UNION ALL SELECT 'supplier', name FROM suppliers ORDER BY 1, 2`. *(Why `UNION ALL` and not `UNION` here?)*

## Expected results (check yourself)

<details>
<summary>Reveal after attempting</summary>

- Task 1: **5** rows — Ada's `manager_id` is `NULL`, so the inner self-join finds no matching manager row and drops her.
- Task 2: **6** rows — Ada now shows `(none — top of chart)`.
- Task 3: the `<` keeps each unordered pair once and prevents pairing an employee with itself. Without it you'd get self-pairs (Ben-Ben) and both orderings (Ben-Ada and Ada-Ben).
- Task 5: `UNION` → **4** rows `{1,2,3,4}`.
- Task 6: `INTERSECT` → **3** rows `{1,2,3}`.
- Task 7: `suppliers EXCEPT customers` → **1** row `{4}`. The reverse (`customers EXCEPT suppliers`) → **0** rows, because every customer region (1,2,3) also has a supplier.
- Task 8: `UNION` → **4** rows; `UNION ALL` → **10** rows. Customers contribute 6 non-null region values (Acme, Bolt Co, Cog Ltd, Dyeworks, Echo LLC, Gizmo Inc — Foxtrot's NULL is filtered out); suppliers contribute 4; total 10 occurrences. `UNION` collapses them to the 4 distinct values `{1,2,3,4}`; `UNION ALL` keeps all 10.
- Task 9: `UNION ALL` — a customer and a supplier could share a name, and even if not, you want *every* party listed, not a de-duplicated set. `UNION` would also do needless de-dup work.

</details>

## Done when…

- [ ] `solutions.sql` has all nine queries with row-count comments.
- [ ] Your self-joins use two distinct aliases and you can explain why that's required.
- [ ] You can state when `EXCEPT` is cleaner than a `LEFT JOIN … IS NULL` (comparing keys only) and when the join wins (needing extra columns).
- [ ] You explained the `UNION` vs `UNION ALL` count difference.

## Stretch

- Rewrite task 7 (`suppliers EXCEPT customers`) as an anti-join (`NOT EXISTS`) returning the region *name*, not just the id. Which phrasing is clearer? (This is the "set vs join" judgment from Lecture 3.)
- Build a query listing every employee with their **depth** in the org chart (CEO = 0, direct reports = 1, …) using nested self-joins up to 3 levels. (Recursive CTEs do this properly — a Week 3 preview.)

## Submission

Commit `solutions.sql` to your portfolio under `c33-week-02/exercise-03/`.
