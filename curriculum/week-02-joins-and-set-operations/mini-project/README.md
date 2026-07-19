# Mini-Project — Reconstruct a Report Across 5 Joined Tables

> Build one report that stitches together the whole `crunch_shop` schema: a **regional sales summary** that only exists once you join customers, orders, order items, products, and regions. This is the week's capstone — every join skill in one deliverable.

**Estimated time:** 3–5 hours, Friday–Saturday.

This mini-project produces a small set of SQL files and a short write-up. The goal is not a single clever query but a **coherent report built from at least five joined tables**, delivered with the reasoning behind your join choices. It's the Week-2 deliverable because it forces you to do the real thing: fix a grain, chain joins across a schema, decide `LEFT` vs `INNER` deliberately, and defend it.

Load `crunch_shop` first — see [exercises/README.md](../exercises/README.md).

---

## Deliverable

A directory in your portfolio repo `c33-week-02/mini-project/` containing:

1. `report.sql` — the main report query (the five-table one, below).
2. `supporting.sql` — the three supporting queries (defined below).
3. `REPORT.md` — the rendered output of each query (paste the result tables) plus a written analysis.
4. `NOTES.md` — your join-design decisions.

---

## The report: "Regional Sales Summary"

The stakeholder question: **"For each region, which products sold, to which customers, and what did each line contribute?"** That single question spans five tables — `regions`, `customers`, `orders`, `order_items`, `products` — and cannot be answered from fewer.

### Main query requirements (`report.sql`)

One `SELECT` returning **one row per sold order line**, with these columns in order:

| Column | Source | Notes |
|--------|--------|-------|
| `region` | `regions.name` | `COALESCE` to `'(no region)'` for region-less customers |
| `customer` | `customers.name` | |
| `order_id` | `orders.order_id` | |
| `order_date` | `orders.order_date` | |
| `product` | `products.name` | |
| `quantity` | `order_items.quantity` | |
| `unit_price` | `order_items.unit_price` | |
| `line_total` | computed | `quantity * unit_price` |

Rules:

- **Grain:** one row per `order_items` row. Only *sold* lines appear (an order line implies a real sale), so the driving table is `order_items` and the joins outward to orders/customers/products are `INNER` — a line always has all four. The join to `regions` must be `LEFT` (Foxtrot has no region).
- Exclude `cancelled` orders (`WHERE o.status <> 'cancelled'`). Note in `NOTES.md` how many rows that removes and why a `WHERE` on `orders.status` is safe here (it's the *driving* side, not an outer partner).
- Sort by `region`, then `customer`, then `order_id`, then `product`.

### Supporting queries (`supporting.sql`)

Three shorter queries that use the *other* skills of the week:

1. **Coverage grid (cross join + anti-join).** Using `regions CROSS JOIN categories` as a template, list every region×category pair that has **no** sold product — the gaps in the catalog's regional coverage. *(Cross join to build the grid, `LEFT JOIN`/`NOT EXISTS` to find the empty cells.)*

2. **Dormant + dead stock (set operation).** Produce one two-column list, tagged by `reason`, combining: customers who never ordered (`reason = 'dormant customer'`) and products never sold (`reason = 'dead stock'`). Use `UNION ALL`. *(This is the anti-join + set-op combination from Lectures 2 and 3.)*

3. **Org roll-up (self-join).** For each sales rep who took at least one non-cancelled order, show the rep's name and their manager's name (self-join on `employees`), plus a count of orders they handled. *(You'll need `GROUP BY` — a light Week-3 preview; `count(*)` per rep is enough.)*

---

## Milestones

Work in this order — each stage is runnable before the next.

### Milestone 1 — Two tables (30 min)
Get `order_items` joined to `orders` returning line-level rows with `line_total`. Confirm the grain: 10 lines total, fewer once you exclude cancelled.

### Milestone 2 — Add customer + product (45 min)
Chain in `customers` and `products`. You now have names on every line. Still `INNER` — verify no rows dropped.

### Milestone 3 — Add region with a LEFT join (45 min)
Join `customers → regions` as `LEFT` and `COALESCE` the name. Confirm Foxtrot's line shows `(no region)` and was **not** dropped. This is the join that proves you understood the week.

### Milestone 4 — Filter, sort, finalize (30 min)
Add the `cancelled` exclusion and the sort. Lock down `report.sql`. Paste the output into `REPORT.md`.

### Milestone 5 — The three supporting queries (60–90 min)
Cross-join grid, the `UNION ALL` hygiene list, the self-join roll-up. Paste outputs.

### Milestone 6 — Write it up (45 min)
`NOTES.md`: for the main query, justify each `LEFT` vs `INNER`; for the supporting queries, say which technique each demonstrates. `REPORT.md`: 150–250 words interpreting the numbers (which region sold most, what's dead stock, who's dormant).

---

## Acceptance criteria

- [ ] `report.sql` joins **at least five tables** and returns one row per sold, non-cancelled order line with all eight columns and a correct `line_total`.
- [ ] The region join is `LEFT` + `COALESCE`; Foxtrot's line survives as `(no region)`.
- [ ] Cancelled orders are excluded (order 105 and its Nut line disappear).
- [ ] `supporting.sql` contains all three queries, each using a *different* Week-2 skill (cross join/anti-join, set op, self-join).
- [ ] `REPORT.md` shows every query's actual output plus a written interpretation.
- [ ] `NOTES.md` justifies every `LEFT` vs `INNER` choice.

---

## Rubric

| Criterion | Weight | "Great" looks like |
|-----------|------:|--------------------|
| Correctness | 30% | Right rows, right `line_total`, right grain; cancelled excluded |
| Join design | 25% | `LEFT` vs `INNER` chosen deliberately and justified; nothing dropped by accident |
| Breadth | 20% | Supporting queries genuinely exercise cross/anti/self-joins + a set op |
| Readability | 15% | Aliased, laddered, sorted; a stranger could read it |
| Analysis | 10% | `REPORT.md` interpretation is specific, drawn from the actual numbers |

---

## Reference: what the main report should roughly look like

After excluding the cancelled order, you should see order lines grouped by region — North customers (Acme, Bolt Co) and their Bolt/Nut/Cog lines, South customers (Cog Ltd, Echo LLC), East (Dyeworks), and Foxtrot under `(no region)`. Nine sold lines survive (10 total minus the one cancelled Nut line in order 105). If your count is off, re-check the grain and the `cancelled` filter before anything else.

---

When done: push, then take the [quiz](../quiz.md). Next up — [Week 3 — Aggregation, subqueries & CTEs](../../week-03-aggregation-subqueries-and-ctes/), where you'll `GROUP BY` the rows you just learned to join.
