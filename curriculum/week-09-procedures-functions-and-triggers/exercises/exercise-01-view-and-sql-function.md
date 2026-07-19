# Exercise 1 — A View and a SQL Function

**Goal:** Name a repeated query with a view, then wrap a scalar calculation in a `LANGUAGE sql` function and use both together.

**Estimated time:** 40 minutes. **Prereq:** the practice schema from [the exercises README](./README.md).

## Part A — a view

You keep writing "paid orders with their customer's name and country." Name it once.

### Steps

1. Create a view `paid_orders` that returns `order_id`, `customer_id`, `full_name`, `country`, and `placed_at` for every order whose `status = 'paid'`. Join `orders` to `customers`.

```sql
CREATE VIEW paid_orders AS
SELECT o.order_id,
       o.customer_id,
       c.full_name,
       c.country,
       o.placed_at
FROM   orders o
JOIN   customers c ON c.customer_id = o.customer_id
WHERE  o.status = 'paid';
```

2. Query it to confirm it works:

```sql
SELECT country, count(*) AS paid_orders
FROM   paid_orders
GROUP  BY country
ORDER  BY paid_orders DESC;
```

**Expected:** `US` has 3 paid orders, `GB` has 1. (Order 4 is `pending`, so it's excluded.)

3. Inspect the stored definition:

```sql
SELECT pg_get_viewdef('paid_orders', true);
```

## Part B — a SQL function

Now a reusable scalar: the total value of an order in cents.

### Steps

4. Create a `LANGUAGE sql` function `order_total_cents(bigint)` that sums `quantity * unit_price_cents` for a given `order_id`. Mark it `STABLE` (it reads the DB but doesn't write). Use `coalesce` so an order with no items returns `0`, not `NULL`.

```sql
CREATE OR REPLACE FUNCTION order_total_cents(p_order_id bigint)
RETURNS bigint
LANGUAGE sql
STABLE
AS $$
    SELECT coalesce(sum(quantity * unit_price_cents), 0)
    FROM   order_items
    WHERE  order_id = p_order_id;
$$;
```

5. Call it on a single order:

```sql
SELECT order_total_cents(1);   -- Keyboard 7999 + Mouse 2*2499 = 12997
```

**Expected:** `12997`.

## Part C — combine them

6. Use the function *and* the view together to list each paid order with its dollar total, biggest first:

```sql
SELECT p.order_id,
       p.full_name,
       order_total_cents(p.order_id) / 100.0 AS total_dollars
FROM   paid_orders p
ORDER  BY total_dollars DESC;
```

**Expected (first row):** order 2 (Ada, Monitor) at `199.99`.

## Questions (answer in `notes.md`)

1. Does `paid_orders` take any storage? Explain in one sentence what the database actually stored.
2. Why is `order_total_cents` marked `STABLE` and not `IMMUTABLE`? What would break if you marked it `IMMUTABLE`?
3. If you `INSERT` a new paid order right now, does `paid_orders` show it immediately? Why?

## Done when…

- [ ] `paid_orders` returns 4 rows and the country counts match.
- [ ] `order_total_cents(1)` returns `12997`.
- [ ] Part C lists paid orders sorted by dollar total.
- [ ] All three questions answered in `notes.md`.

## Stretch

- Rewrite `order_total_cents` to return `numeric` dollars instead of `bigint` cents, and update Part C.
- Add a second view `order_summary` that includes the total directly (call the function inside the view). What are the pros and cons of putting the function call *in* the view?
