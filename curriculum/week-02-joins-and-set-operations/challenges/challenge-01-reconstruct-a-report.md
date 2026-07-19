# Challenge 1 — Reconstruct a Report

**Time:** ~75 minutes. **Difficulty:** Medium.

## Scenario

Sales pulled this report last quarter and then lost the query that made it. All they kept is the output — a per-order-line breakdown they need again for every future quarter. Your job: reconstruct the query from the output alone, against `crunch_shop`.

Here is the report they saved (a few representative rows shown; yours should reproduce **every** order line):

```
 order_id | order_date | customer | region | rep | category      | product  | qty | unit_price | line_total
----------+------------+----------+--------+-----+---------------+----------+-----+------------+-----------
      101 | 2026-01-05 | Acme     | North  | Ben | Widgets       | Bolt     |  10 |       2.50 |      25.00
      101 | 2026-01-05 | Acme     | North  | Ben | Widgets       | Nut      |  20 |       1.25 |      25.00
      104 | 2026-03-01 | Cog Ltd  | South  | Eve | Gadgets       | Cog      |   3 |       9.00 |      27.00
      104 | 2026-03-01 | Cog Ltd  | South  | Eve | Gadgets       | Sprocket |   1 |      12.00 |      12.00
      107 | 2026-04-10 | Foxtrot  | (none) | Ben | Gadgets       | Sprocket |   2 |      12.00 |      24.00
      ...
```

## The task

Write one query that produces this report. Requirements the saved output implies:

1. **One row per order line** (per `order_items` row). The grain is the line item, not the order.
2. Columns, in this order: `order_id`, `order_date`, `customer`, `region`, `rep`, `category`, `product`, `qty`, `unit_price`, `line_total` — where `line_total = quantity * unit_price`.
3. `region` must show `(none)` for a customer with no region (look at the Foxtrot row).
4. **Every** order line must appear — including lines whose product has no category, and orders by customers with no region. Nothing may be silently dropped.
5. Sort by `order_id`, then `product`.

## Constraints

- One `SELECT` statement. No temp tables, no application code.
- You'll need **five** base tables at least: `order_items`, `orders`, `customers`, `employees`, `products` — plus `categories` and `regions` for the name columns (so really six/seven). Decide which joins must be `LEFT` and which can be `INNER`.
- Qualify every column with a table alias.

## Hints

<details>
<summary>Choosing the grain and the driving table</summary>

The report has one row per *line item*, so start `FROM order_items oi` and join outward. Everything else (`orders`, then `customers`/`employees`/`products`, then `regions`/`categories`) hangs off that.

</details>

<details>
<summary>Which joins must be LEFT?</summary>

Requirement 4 is the whole trick. Product 6 (Gizmo-Pro) has no category and customer 6 (Foxtrot) has no region. If you `INNER JOIN` to `categories` or `regions`, those rows vanish. Any join reaching a table that can be *absent* must be `LEFT`. The joins along mandatory foreign keys (`order_items → orders`, `order_items → products`, `orders → customers`, `orders → employees`) can be `INNER` — but making them `LEFT` too is a defensible "never drop a line" stance. Explain your choice.

</details>

<details>
<summary>The (none) region</summary>

`COALESCE(r.name, '(none)')`. Same pattern shows up for category if any ordered product were uncategorized.

</details>

## How success is judged

- [ ] The result reproduces every order line in `crunch_shop` (10 lines) with all ten columns.
- [ ] `line_total` is correct and equals `quantity * unit_price` per row.
- [ ] Foxtrot's order (107) appears with region `(none)` — proving your region join is `LEFT` + `COALESCE`.
- [ ] No line is dropped for a missing category or region.
- [ ] The query is one readable statement, aliased and sorted as specified.
- [ ] `NOTES.md` explains: which joins you made `LEFT` and why, and one alternative join order you considered.

## Stretch

- Add a `supplier` column (product → supplier). What happens to any product with a `NULL` supplier — and does that change which join must be `LEFT`?
- Add an order-level `order_total` column (sum of that order's line totals) *without* changing the grain. *(You'll need a subquery or window function — a genuine Week 3/Week 8 preview; try it and see where pure joins run out.)*

## Submission

Commit `report.sql` + `NOTES.md` to your portfolio under `c33-week-02/challenge-01/`.

## Why this matters

Reconstructing a report from its output is a real, weekly task on any data team — the query is lost, the numbers are trusted, and you have to rebuild it exactly. It forces the two skills that matter most this week: fixing the **grain** of a result, and knowing precisely which joins may drop rows.
