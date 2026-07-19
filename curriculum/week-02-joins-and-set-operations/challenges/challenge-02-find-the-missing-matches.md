# Challenge 2 — Find the Missing Matches

**Time:** ~60 minutes. **Difficulty:** Medium.

## Scenario

The business wants a "data hygiene" report: everything in `crunch_shop` that is **stranded** — a row that *should* have a partner somewhere but doesn't. Dormant customers, dead-stock products, suppliers you're paying for nothing, orders with no line items. Each is an **anti-join**, and your job is to find all of them, phrase each one twice, and decide which phrasing you'd ship.

## The task

Produce a set of queries — one per question below — each returning the stranded rows. For **at least three** of them, write the query **two ways** (`NOT EXISTS` *and* `LEFT JOIN … IS NULL`, or `EXCEPT`), confirm the two agree, and note which you'd keep.

Find:

1. **Customers who have never placed an order.**
2. **Products that have never been ordered** (never appear in `order_items`).
3. **Suppliers that supply no products.**
4. **Categories that contain no product** *(is there one? report honestly — an empty result is a valid answer).*
5. **Employees who have taken no order** (never appear as `orders.employee_id`).
6. **Regions that have no customer.**
7. **The hard one:** orders that have **no** line items in `order_items`. *(In the shipped seed every order has items — so this should return zero rows. Prove it, then insert one item-less order, re-run, and show it appears. Clean up after.)*

## Constraints

- Return enough columns to make each stranded row identifiable (id **and** name), not just the id.
- For the three you phrase twice, the two versions must return **identical** rows.
- No hard-coded id lists. The queries must keep working if the data changes.

## Hints

<details>
<summary>The two anti-join shapes</summary>

```sql
-- Shape A — NOT EXISTS (preferred; NULL-safe, returns any columns of A):
SELECT a.id, a.name
FROM a
WHERE NOT EXISTS (SELECT 1 FROM b WHERE b.a_id = a.id);

-- Shape B — LEFT JOIN ... IS NULL (test a NON-nullable right column, e.g. a PK):
SELECT a.id, a.name
FROM a
LEFT JOIN b ON b.a_id = a.id
WHERE b.id IS NULL;
```

</details>

<details>
<summary>Why not NOT IN?</summary>

`WHERE a.id NOT IN (SELECT b.a_id FROM b)` looks equivalent and is a trap: if `b.a_id` ever contains a `NULL`, the whole query returns **zero rows** (because `x NOT IN (…, NULL)` is never true). Use `NOT EXISTS`. If you *do* demo `NOT IN`, show the failure it can cause.

</details>

<details>
<summary>For question 7, to create then remove a test order</summary>

```sql
INSERT INTO orders (order_id, customer_id, employee_id, order_date, status)
VALUES (999, 1, 2, DATE '2026-05-01', 'pending');   -- order with NO items
-- run your anti-join; order 999 should appear
DELETE FROM orders WHERE order_id = 999;            -- clean up
```

</details>

## Expected findings (peek only after trying)

<details>
<summary>Reveal</summary>

- Q1: **Gizmo Inc** (id 7).
- Q2: **Widget-X** (id 5) — and note Gizmo-Pro (id 6) is *also* never ordered, so **two** products: 5 and 6. (Did your query catch both? Gizmo-Pro has a NULL category, which is a good reminder that "never ordered" is about `order_items`, not categories.)
- Q3: **WestParts** (id 4).
- Q4: **none** — every category (Widgets, Gadgets, Gizmos) has at least one product. An empty result is the correct answer; report it as such.
- Q5: **Finn** (id 6) — and Ada (id 1) never took an order either, so **two** employees: Ada and Finn.
- Q6: **West** (id 4).
- Q7: **zero** in the shipped seed; **order 999** after the test insert.

</details>

## How success is judged

- [ ] All seven questions answered, each returning identifiable rows (id + name).
- [ ] At least three questions phrased two ways, with a note that the results match.
- [ ] Question 2 catches **both** never-ordered products (5 and 6), and question 5 catches **both** order-less employees (Ada and Finn) — a query that misses one is subtly wrong.
- [ ] Question 4's empty result is reported honestly, not hidden.
- [ ] `NOTES.md` states, for each doubled query, which phrasing you'd ship and why (readability, NULL-safety, or columns needed).

## Stretch

- Combine several of these into **one** hygiene report using `UNION ALL` with a `reason` label column: `'customer never ordered'`, `'product never sold'`, etc.
- Which of your anti-joins could be written as `EXCEPT`, and which *can't* (because you need columns beyond the key)? List them.

## Submission

Commit `hygiene.sql` + `NOTES.md` to your portfolio under `c33-week-02/challenge-02/`.

## Why this matters

"Rows with no partner" is one of the highest-value queries in real work — churned users, unsold inventory, orphaned records, broken foreign keys after a bad migration. Master the anti-join and you can audit any schema on demand.
