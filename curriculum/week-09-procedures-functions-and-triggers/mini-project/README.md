# Mini-Project — An Audited Table with Triggers and Functions

> Build a small but complete piece of database machinery: a table whose every change is audited automatically, whose derived columns stay correct without trusting the writer, and whose common operations are exposed as functions. This is the week distilled into one artifact.

**Estimated time:** 5 hours, spread across Friday–Saturday. **Engine:** PostgreSQL 16.

This is the deliverable the syllabus promises for Week 9: *"Ship an audited table with triggers + functions."* You'll combine everything — a view, a materialized view, SQL and PL/pgSQL functions, and triggers — into a coherent little system you could actually drop into a real schema.

---

## The domain: a simple inventory ledger

You're building the inventory core for a small warehouse app. Two tables:

- `products` — the catalog, with a **maintained** `stock_qty` derived column.
- `stock_movements` — an append-only ledger of every stock in/out event.

The invariant: **`products.stock_qty` must always equal the sum of that product's `stock_movements.delta`** — and it must stay correct no matter who inserts a movement, without the app ever setting `stock_qty` by hand. Every change to `products` must be audited.

---

## Requirements

### 1. Schema

```sql
CREATE TABLE products (
    product_id  bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku         text NOT NULL UNIQUE,
    name        text NOT NULL,
    stock_qty   int  NOT NULL DEFAULT 0,   -- maintained by trigger, never set by hand
    updated_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE stock_movements (
    movement_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_id  bigint NOT NULL REFERENCES products(product_id),
    delta       int    NOT NULL CHECK (delta <> 0),   -- +receipt / -shipment
    reason      text   NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now()
);
```

### 2. A generic audit log + trigger (Lecture 3, Pattern A)

- An `audit_log` table capturing `table_name`, `operation`, `row_pk`, `old_row jsonb`, `new_row jsonb`, `changed_by`, `changed_at`.
- One generic trigger function (using `TG_OP`, `TG_TABLE_NAME`, `to_jsonb`) bound to `products` for INSERT/UPDATE/DELETE.

### 3. A derived-column trigger (Lecture 3, Pattern B)

- An `AFTER INSERT ON stock_movements` trigger that updates the parent product's `stock_qty` by `NEW.delta`. (It must be `AFTER` and touch *another* table — this is the legitimate cross-table case a generated column can't handle.)
- The trigger must also **refuse** any movement that would drive `stock_qty` negative (raise an exception — you can't ship what you don't have).

### 4. Functions (Lecture 2)

- `receive_stock(p_sku text, p_qty int, p_reason text)` — a PL/pgSQL function that looks up the product by SKU, inserts a positive movement, and returns the new `stock_qty`. Raise `no_data_found` if the SKU doesn't exist; reject `p_qty <= 0`.
- `ship_stock(p_sku text, p_qty int, p_reason text)` — same, inserting a negative movement. Let the derived-column trigger's negative-stock guard do the enforcement, and catch it to return a clean message.
- `product_stock(p_sku text)` — a `LANGUAGE sql` `STABLE` function returning the current `stock_qty` for a SKU.

### 5. A reporting view + materialized view (Lecture 1)

- A view `low_stock` listing products with `stock_qty < 10`.
- A materialized view `daily_movement_summary` (per day, per product: total in, total out, net) with a `UNIQUE` index so it can refresh `CONCURRENTLY`.

---

## Milestones

Work in this order — each builds on the last.

1. **Schema up** (30m): create the four tables; insert 3 products.
2. **Audit trigger** (45m): generic function + binding on `products`; prove an `UPDATE` to a product logs to `audit_log`.
3. **Derived-column trigger** (60m): `stock_qty` maintenance on `stock_movements` insert; test that it tracks the ledger sum and blocks negative stock.
4. **Functions** (75m): `receive_stock`, `ship_stock`, `product_stock`; test happy paths and every error.
5. **Reporting** (45m): `low_stock` view + `daily_movement_summary` mview + unique index + one `REFRESH ... CONCURRENTLY`.
6. **Integrity proof** (30m): the acceptance script below must pass end-to-end.
7. **Write-up** (15m): a short `README.md` explaining each trigger and function and *why each is the right tool* (or where a constraint/generated column would've been better).

---

## Acceptance script

Your system must produce these results:

```sql
-- receive 100 of SKU 'A1', ship 30, expect stock_qty = 70
SELECT receive_stock('A1', 100, 'initial receipt');   -- 100
SELECT ship_stock('A1', 30, 'order #1');              -- 70
SELECT product_stock('A1');                            -- 70

-- stock_qty must equal the ledger sum
SELECT p.stock_qty,
       (SELECT sum(delta) FROM stock_movements m WHERE m.product_id = p.product_id) AS ledger_sum
FROM   products p WHERE p.sku = 'A1';                  -- both 70

-- over-shipping is refused
SELECT ship_stock('A1', 999, 'oops');                  -- clean error, stock unchanged
SELECT product_stock('A1');                            -- still 70

-- audit captured the stock_qty changes on products
SELECT operation, new_row ->> 'stock_qty' FROM audit_log
WHERE  table_name = 'products' ORDER BY audit_id;
```

---

## Rubric

| Criterion | Weight | "Great" looks like |
|-----------|-------:|--------------------|
| Derived column correctness | 25% | `stock_qty` always equals the ledger sum, enforced by trigger, never set by app |
| Audit completeness | 20% | Every `products` change logged with old/new JSON; can't be bypassed |
| Function design | 20% | Clean signatures, right language per function, real error handling |
| Negative-stock guard | 15% | Over-ship refused with a clear message; stock unchanged after |
| Views + mview | 10% | `low_stock` correct; mview refreshes `CONCURRENTLY` |
| Tool judgement | 10% | Write-up explains why each mechanism was chosen over the alternative |

---

## Why this matters

This is the shape of real database logic. Backend services (C16) lean on exactly this: a table that maintains its own invariants and records its own history, so no application bug can corrupt it silently. When Week 10 adds roles and row-level security on top, you'll be securing a table that already defends itself.

---

When done: commit to `c33-week-09/mini-project/`, then take the [quiz](../quiz.md) and move to [Week 10 — Security & Data Integrity](../../week-10-security-and-data-integrity/).
