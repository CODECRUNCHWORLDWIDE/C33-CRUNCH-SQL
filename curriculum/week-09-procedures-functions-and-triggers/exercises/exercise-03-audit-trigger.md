# Exercise 3 — An Audit Trigger

**Goal:** Wire an `AFTER ROW` trigger that records every change to `customers` into an append-only audit log — capturing the operation, the old and new rows, who made the change, and when.

**Estimated time:** 50 minutes. **Prereq:** the practice schema from [the exercises README](./README.md).

## Why an `AFTER ROW` trigger

You want to record changes that *actually happened*, generically, for every writer — the classic case for `AFTER ... FOR EACH ROW` from Lecture 3. `BEFORE` is wrong here (the change isn't final yet); statement-level is wrong (you want per-row detail).

## Steps

1. Create the audit table:

```sql
CREATE TABLE customer_audit (
    audit_id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    operation  text        NOT NULL,
    customer_id bigint,
    old_row    jsonb,
    new_row    jsonb,
    changed_by text        NOT NULL DEFAULT current_user,
    changed_at timestamptz NOT NULL DEFAULT now()
);
```

2. Write the trigger function. Use `TG_OP` to branch and `to_jsonb` to snapshot the rows generically:

```sql
CREATE OR REPLACE FUNCTION audit_customer_change()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO customer_audit(operation, customer_id, old_row, new_row)
    VALUES (
        TG_OP,
        coalesce(NEW.customer_id, OLD.customer_id),
        CASE WHEN TG_OP IN ('UPDATE','DELETE') THEN to_jsonb(OLD) END,
        CASE WHEN TG_OP IN ('INSERT','UPDATE') THEN to_jsonb(NEW) END
    );
    RETURN NULL;   -- AFTER trigger: return value is ignored
END;
$$;
```

3. Bind it for all three write operations:

```sql
CREATE TRIGGER trg_customer_audit
    AFTER INSERT OR UPDATE OR DELETE ON customers
    FOR EACH ROW
    EXECUTE FUNCTION audit_customer_change();
```

## Exercise the trigger

4. Make one of each kind of change:

```sql
INSERT INTO customers(full_name, email, country)
    VALUES ('Alan Turing','alan@example.com','GB');

UPDATE customers SET country = 'DE' WHERE email = 'alan@example.com';

DELETE FROM customers WHERE email = 'alan@example.com';
```

5. Read the audit trail:

```sql
SELECT audit_id, operation, customer_id,
       old_row ->> 'country' AS old_country,
       new_row ->> 'country' AS new_country,
       changed_by, changed_at
FROM   customer_audit
ORDER  BY audit_id;
```

**Expected:** three rows — an `INSERT` (old_country `NULL`, new `GB`), an `UPDATE` (old `GB`, new `DE`), and a `DELETE` (old `DE`, new_country `NULL`).

## Prove it can't be bypassed

6. This is the whole point of a trigger. Change a row a *different* way and confirm the audit still fires:

```sql
UPDATE customers SET status = 'inactive' WHERE customer_id = 1;
SELECT operation, new_row ->> 'status' FROM customer_audit ORDER BY audit_id DESC LIMIT 1;
```

**Expected:** an `UPDATE` row with new status `inactive` — even though you never touched the audit table yourself.

## Questions (answer in `notes.md`)

1. Why `RETURN NULL` here, and would returning `NEW` change anything for this `AFTER` trigger?
2. What does `to_jsonb(OLD)` buy you over listing every column by hand?
3. You bulk-update 100,000 customers in one statement. How many times does this trigger's function run, and what would you consider instead for that path? (Hint: Lecture 3, transition tables.)
4. The audit records `current_user`. In an app where everyone connects as one database role, what does that column actually tell you — and what would you need to capture the *real* end user?

## Done when…

- [ ] `customer_audit` has an INSERT, UPDATE, and DELETE row after step 4.
- [ ] The `status` update in step 6 also produced an audit row.
- [ ] All four questions answered in `notes.md`.

## Stretch

- Make the function **generic** so the *same* function audits `orders` too: write to a shared `audit_log(table_name, ...)` table using `TG_TABLE_NAME`, then bind the one function to both tables.
- Add a `WHEN (OLD.* IS DISTINCT FROM NEW.*)` clause to the `UPDATE` trigger so no-op updates don't create audit noise. Verify an `UPDATE` that changes nothing writes no audit row.
