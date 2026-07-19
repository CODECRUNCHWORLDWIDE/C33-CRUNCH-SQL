# Week 9 — Homework

Six practice tasks (~5 hours total) that reinforce views, functions, and triggers. Do them against the practice schema from the [exercises README](./exercises/README.md), extending it as noted. Save each task's SQL and a one-line answer where asked; commit to `c33-week-09/homework/`.

These are lighter than the mini-project and heavier than the exercises — practice reps for fluency.

---

## Task 1 — A view you'll actually reuse (~30m)

Create a view `customer_order_counts(customer_id, full_name, order_count, paid_count)` that, for every customer, shows their total order count and how many are `paid`. Use conditional aggregation (`count(*) FILTER (WHERE status = 'paid')`).

- Confirm a customer with zero orders still appears (hint: `LEFT JOIN`).
- **Answer:** does this view need `WITH CHECK OPTION`? Why or why not?

## Task 2 — SQL vs PL/pgSQL, same job (~40m)

Write `customer_lifetime_cents(p_customer_id bigint)` returning the sum of `order_total_cents` across all that customer's **paid** orders.

- Write it once as `LANGUAGE sql`.
- Then write `customer_lifetime_cents_v2` as `LANGUAGE plpgsql` doing the same thing with a `RETURN QUERY`-free scalar `SELECT ... INTO`.
- **Answer:** which should ship, and why? Which volatility marker did you use and why?

## Task 3 — Control flow + `RETURNS TABLE` (~50m)

Write `customers_by_tier()` returning `TABLE(customer_id bigint, full_name text, lifetime_cents bigint, tier text)` where `tier` is:

- `'gold'` if lifetime ≥ $200,
- `'silver'` if ≥ $100,
- `'bronze'` otherwise.

Use a `CASE` expression for the tier. Order gold first.

- **Answer:** could you have done this entirely in a plain view with no function? If so, what does the function buy you here (or not)?

## Task 4 — Exception handling (~40m)

Write `safe_close_account(p_customer_id bigint)` that sets a customer's `status` to `'closed'`, but:

- raises `no_data_found` if the customer doesn't exist,
- raises an error if the customer has any `pending` orders (you can't close an account mid-order),
- returns the text `'closed'` on success.

Then write a wrapper that catches both errors and returns a friendly string.

- **Answer:** what happens to the `UPDATE` if the "pending orders" check raises *after* the update in your ordering? Reorder if needed and explain.

## Task 5 — A derived-column trigger vs. a generated column (~40m)

Add a `line_total_cents` column to `order_items`.

- First implement it with a `BEFORE INSERT OR UPDATE` trigger (`NEW.line_total_cents := NEW.quantity * NEW.unit_price_cents`).
- Then drop the trigger and reimplement it as a `GENERATED ALWAYS AS (...) STORED` column.
- **Answer:** verify both give identical results on an `UPDATE` to `quantity`. Which is better here, and what's the *one* situation where you'd be forced back to the trigger?

## Task 6 — An audit trigger with a no-op guard (~40m)

Put a generic audit trigger on `orders` (log INSERT/UPDATE/DELETE to a shared `audit_log`).

- Add `WHEN (OLD.* IS DISTINCT FROM NEW.*)` to the UPDATE path so updates that change nothing don't create audit rows.
- Run an `UPDATE orders SET status = status WHERE order_id = 1;` (a no-op) and confirm **no** audit row appears.
- Then a real status change and confirm one **does**.
- **Answer:** why can't you put the `WHEN (...)` clause on an `AFTER INSERT` trigger the same way? (What is `OLD` on an insert?)

---

## Submission checklist

- [ ] Six `.sql` files (or one file with clear section headers).
- [ ] Every "Answer" question answered in a `homework-answers.md`.
- [ ] Everything runs top-to-bottom against a fresh copy of the practice schema.

## Grading note

Homework is honor-graded — tick it complete in the reader. But Task 5 and Task 6's "Answer" questions are exactly the kind of judgement the Week 9 quiz tests. Don't skip the writing.
