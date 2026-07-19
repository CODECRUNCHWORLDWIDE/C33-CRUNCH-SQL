# Exercise 2 — A PL/pgSQL Function with Control Flow and Error Handling

**Goal:** Write a `LANGUAGE plpgsql` function that branches, reads the database into variables, raises a proper error, and catches it.

**Estimated time:** 50 minutes. **Prereq:** the practice schema + `order_total_cents` from Exercise 1.

## The task

Write `place_order_discount(p_order_id bigint, p_pct numeric)` that returns the discounted dollar total of an order, applying these rules:

- If the order doesn't exist, raise an error (`no_data_found`).
- If `p_pct` is not between `0` and `100`, raise an error (`invalid_parameter_value`).
- Orders over $150 get an **extra** 5 percentage points off (a loyalty bump), on top of `p_pct`, but the total discount is capped at 100%.
- Return the discounted total as `numeric` dollars, rounded to 2 places.

This needs a variable, a lookup, `IF` branching, a `RAISE`, and arithmetic — exactly the muscles from Lecture 2.

## Steps

1. Write the function skeleton with a `DECLARE` block:

```sql
CREATE OR REPLACE FUNCTION place_order_discount(p_order_id bigint, p_pct numeric)
RETURNS numeric
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
    v_total_cents bigint;
    v_total       numeric;
    v_pct         numeric := p_pct;
BEGIN
    -- validate p_pct
    IF p_pct < 0 OR p_pct > 100 THEN
        RAISE EXCEPTION 'discount pct % out of range 0..100', p_pct
            USING ERRCODE = 'invalid_parameter_value';
    END IF;

    -- confirm the order exists and get its total
    IF NOT EXISTS (SELECT 1 FROM orders WHERE order_id = p_order_id) THEN
        RAISE EXCEPTION 'order % not found', p_order_id
            USING ERRCODE = 'no_data_found';
    END IF;

    v_total_cents := order_total_cents(p_order_id);
    v_total       := v_total_cents / 100.0;

    -- loyalty bump for big orders
    IF v_total > 150 THEN
        v_pct := least(v_pct + 5, 100);
    END IF;

    RETURN round(v_total * (1 - v_pct / 100.0), 2);
END;
$$;
```

2. Test the happy path:

```sql
SELECT place_order_discount(2, 10);   -- order 2 total = 199.99, >150 so 15% off
```

**Expected:** `169.99` (199.99 × 0.85, rounded).

3. Test the loyalty threshold boundary:

```sql
SELECT place_order_discount(1, 10);   -- order 1 total = 129.97, NOT >150, plain 10%
```

**Expected:** `116.97`.

4. Trigger each error and read the message:

```sql
SELECT place_order_discount(999, 10);   -- no such order
SELECT place_order_discount(1, 150);    -- bad pct
```

Both should raise, with the message you wrote.

## Part B — catch the error

5. Write a wrapper `try_discount(bigint, numeric)` that returns text — the dollar total as a string on success, or a friendly message on failure — using an `EXCEPTION` block:

```sql
CREATE OR REPLACE FUNCTION try_discount(p_order_id bigint, p_pct numeric)
RETURNS text
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN '$' || place_order_discount(p_order_id, p_pct)::text;
EXCEPTION
    WHEN no_data_found THEN
        RETURN 'no such order';
    WHEN invalid_parameter_value THEN
        RETURN 'discount must be 0..100';
END;
$$;
```

6. Confirm it swallows the errors:

```sql
SELECT try_discount(2, 10);     -- '$169.99'
SELECT try_discount(999, 10);   -- 'no such order'
SELECT try_discount(1, 150);    -- 'discount must be 0..100'
```

## Questions (answer in `notes.md`)

1. Why does `try_discount` catch by error-code name (`no_data_found`) instead of matching the message text?
2. What happens to the transaction if `place_order_discount` raises and *nothing* catches it?
3. The loyalty bump uses `least(v_pct + 5, 100)`. What bug would appear if you dropped the `least(...)` cap and passed `p_pct = 98` on a big order?

## Done when…

- [ ] `place_order_discount(2, 10)` returns `169.99` and `place_order_discount(1, 10)` returns `116.97`.
- [ ] Both error cases raise with your messages.
- [ ] `try_discount` returns the three expected strings.
- [ ] All three questions answered.

## Stretch

- Add a `FOR` loop version `discount_all_orders(p_pct numeric)` that returns a `TABLE(order_id bigint, discounted numeric)` for every paid order — using `RETURN QUERY` instead of a loop first, then rewrite it with an explicit loop and `RETURN NEXT`, and note which reads better.
