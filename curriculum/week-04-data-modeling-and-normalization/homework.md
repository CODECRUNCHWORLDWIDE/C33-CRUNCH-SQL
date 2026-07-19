# Week 4 — Homework

Six problems, ~6 hours total. Commit each to your portfolio under `c33-week-04/homework/`. Use PostgreSQL 16 where SQL is required (SQLite is fine if you enable `PRAGMA foreign_keys = ON;` and `STRICT` tables).

---

## Problem 1 — Key taxonomy in your own words (30 min)

In `keys.md`, define — in one or two sentences each, with a *concrete* example — every term: superkey, candidate key, primary key, alternate key, composite key, foreign key, natural key, surrogate key. Then answer: for a `users` table with columns `(id, username, email)` where username and email are both unique, list every candidate key and say which you'd make the PK and why.

**Acceptance:** eight terms defined with examples; the `users` PK choice justified in ≥2 sentences.

---

## Problem 2 — Cardinality drill (45 min)

For each pair below, state the cardinality (1:1, 1:N, M:N), say where the foreign key goes (or that a junction table is needed and name it), and write the `CREATE TABLE`(s) that implement it. Put answers in `cardinality.sql` + comments.

1. A `country` and its `cities`.
2. A `person` and their `passport`.
3. `students` and `clubs` (a student joins many clubs; a club has many students).
4. An `employee` and their `manager` (who is also an employee — a self-reference).

**Acceptance:** all four modeled with runnable DDL; #3 uses a junction table; #4 is a self-referencing FK.

---

## Problem 3 — Normalize by hand (60 min)

Given this table, normalize to 3NF on paper/in Markdown, showing every step:

`invoices(invoice_id, customer_id, customer_name, customer_city, product_code, product_name, unit_price, qty)`

In `p3-normalize.md`: list all functional dependencies, identify the 1NF/2NF/3NF issues (there's a composite-key partial dependency *and* a transitive one), and give the final set of tables. Name one update, one insert, and one delete anomaly the original suffers.

**Acceptance:** FDs listed; each normal-form violation named; final 3NF tables given; three anomalies described.

---

## Problem 4 — Constraint hardening (60 min)

Take this too-permissive table and rewrite it in `p4-hardened.sql` so the database rejects every kind of bad row:

```sql
CREATE TABLE events (
    id         INT,
    title      TEXT,
    starts_at  TEXT,
    ends_at    TEXT,
    capacity   INT,
    organizer_email TEXT,
    visibility TEXT
);
```

Requirements: a proper surrogate PK; `title` and `organizer_email` NOT NULL; `organizer_email` UNIQUE per event is wrong (an organizer runs many events) — instead reference an `organizers` table; `starts_at`/`ends_at` as `TIMESTAMPTZ` with a CHECK that `ends_at > starts_at`; `capacity > 0`; `visibility` restricted to `('public','private','unlisted')`. Then write three `INSERT`s that each fail, one that succeeds, with the error text pasted in.

**Acceptance:** hardened DDL + an `organizers` table + FK; four inserts (3 fail, 1 succeeds) with output.

---

## Problem 5 — `ON DELETE` decisions (45 min)

For the schema below, decide the correct `ON DELETE` action for **each** foreign key, and write one sentence justifying each. Put the reasoning in `p5-ondelete.md` and the final DDL in `p5.sql`.

- `orders.customer_id → customers`
- `order_items.order_id → orders`
- `order_items.product_id → products`
- `employees.manager_id → employees` (self-ref)
- `documents.owner_id → users`

For each, ask: is this a "part-of" (composition → CASCADE) or a "refers-to" (association → RESTRICT/NO ACTION), or is the link optional (→ SET NULL)?

**Acceptance:** five FKs, each with an action + a one-sentence rationale; DDL runs.

---

## Problem 6 — Denormalize on purpose (60 min)

Start from a normalized `orders`/`order_items` pair. In `p6-denormalize.md`:

1. Write the normalized query that computes each order's total by summing its items.
2. Argue *when* you'd add a stored `orders.total_amount` column instead (what read pattern, what scale).
3. Describe exactly how you'd keep it correct (a trigger on `order_items` insert/update/delete — sketch the logic in words; you'll write real triggers in Week 9).
4. State what you *lose* by denormalizing and how you'd document the trade-off.

**Acceptance:** the normalized query; a concrete scenario justifying the denormalization; a sync strategy; an honest cost statement.

---

## Time budget

| Problem | Time |
|--------:|-----:|
| 1 | 30 min |
| 2 | 45 min |
| 3 | 1 h |
| 4 | 1 h |
| 5 | 45 min |
| 6 | 1 h |
| **Total** | **~5 h** |

After homework, ship the [mini-project](./mini-project/README.md).
