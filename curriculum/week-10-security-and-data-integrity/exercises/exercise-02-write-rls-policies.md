# Exercise 2 — Write RLS Policies

**Goal:** Turn a shared, multi-tenant table into one where each tenant provably sees only its own rows. You'll enable and *force* RLS, write a policy with both `USING` and `WITH CHECK`, drive it with a session variable, and run the attack tests that prove isolation holds — including the cross-tenant write that `WITH CHECK` must block.

**Estimated time:** ~2 hours. Do Exercise 1 first (you'll reuse `migrator`, `web_app`, `app_write`).

## Setup

As `migrator`, create a multi-tenant table and seed three tenants:

```sql
SET ROLE migrator;
CREATE TABLE documents (
  id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id  int  NOT NULL,
  title      text NOT NULL,
  deleted_at timestamptz
);
CREATE INDEX ON documents (tenant_id);            -- policy-friendly index
INSERT INTO documents (tenant_id, title) VALUES
  (1,'Acme Q1 plan'), (1,'Acme roadmap'),
  (2,'Globex budget'), (2,'Globex memo'),
  (3,'Initech spec');
RESET ROLE;

-- Let the app role use the table (privilege is still required alongside RLS)
GRANT SELECT, INSERT, UPDATE, DELETE ON documents TO app_write;
```

## Steps

1. **Confirm the leak first.** As `web_app`, run `SELECT * FROM documents;` — you'll see **all five rows across all tenants**. That's the problem you're about to fix.

2. **Enable and force RLS** (as the owner/`migrator` or a superuser):
   ```sql
   ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
   ALTER TABLE documents FORCE  ROW LEVEL SECURITY;   -- apply even to the owner
   ```

3. **Observe fail-closed.** As `web_app`, `SELECT * FROM documents;` now returns **zero rows** — RLS with no policy denies everything. Good; that's the safe default.

4. **Write the isolation policy.** Create one `FOR ALL` policy that reads the current tenant from a session variable, guarding both reads and writes:
   ```sql
   CREATE POLICY tenant_isolation ON documents
     USING      (tenant_id = current_setting('app.tenant_id', true)::int)
     WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::int);
   ```
   Note the two-argument `current_setting(..., true)` so an *unset* variable yields NULL (no rows) instead of an error.

5. **Set the tenant and read.** As `web_app`:
   ```sql
   SET app.tenant_id = '1';
   SELECT * FROM documents;      -- only Acme's 2 rows
   SET app.tenant_id = '2';
   SELECT * FROM documents;      -- only Globex's 2 rows
   ```

6. **Attack test — cross-tenant read.** While `app.tenant_id = '2'`, try to force-read tenant 1:
   ```sql
   SELECT * FROM documents WHERE tenant_id = 1;   -- expect ZERO rows, not Acme's
   ```
   The policy predicate is AND-ed with your `WHERE`; you cannot widen past it.

7. **Attack test — cross-tenant write.** Still as tenant 2, try to plant a row in tenant 1:
   ```sql
   INSERT INTO documents (tenant_id, title) VALUES (1, 'stolen');   -- expect: violates RLS policy
   ```
   Then try to *hand away* one of your own rows:
   ```sql
   UPDATE documents SET tenant_id = 1 WHERE title = 'Globex memo';  -- expect: violates RLS policy
   ```
   Both must fail — that's `WITH CHECK` doing its job.

8. **Prove the index is used.** With a tenant set, confirm the planner folds the predicate in:
   ```sql
   EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM documents;
   ```

9. **Add a RESTRICTIVE rule.** Hide soft-deleted rows from everyone regardless of tenant:
   ```sql
   UPDATE documents SET deleted_at = now() WHERE title = 'Acme roadmap';   -- run as migrator
   CREATE POLICY hide_deleted ON documents AS RESTRICTIVE
     USING (deleted_at IS NULL);
   ```
   As `web_app` with `app.tenant_id = '1'`, you should now see **one** Acme row, not two.

## Expected result

- Before the policy: `web_app` sees 5 rows (leak) → after `ENABLE`: 0 rows → after the policy + `SET`: only the current tenant's rows.
- Cross-tenant read returns nothing; cross-tenant insert/update both raise *"new row violates row-level security policy for table documents."*
- With the RESTRICTIVE `hide_deleted` policy, soft-deleted rows disappear for tenant 1.

## Done when…

- [ ] RLS is `ENABLE`d **and** `FORCE`d on `documents`.
- [ ] A `FOR ALL` policy with both `USING` and `WITH CHECK` isolates by `app.tenant_id`.
- [ ] Unset `app.tenant_id` yields zero rows (fail-closed), not an error.
- [ ] Both cross-tenant write attempts are rejected.
- [ ] The RESTRICTIVE policy removes soft-deleted rows on top of tenant isolation.

## Stretch

- Replace `SET app.tenant_id` with `SET LOCAL app.tenant_id` inside a `BEGIN ... COMMIT` and explain why `LOCAL` is safer behind a connection pool.
- Add a second, read-only login role `auditor` that is granted `BYPASSRLS`, and demonstrate it sees every tenant — then argue in one sentence why that attribute must be tightly controlled.
