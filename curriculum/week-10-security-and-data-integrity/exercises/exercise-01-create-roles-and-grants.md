# Exercise 1 — Create Roles and Grants

**Goal:** Build a three-tier role hierarchy (`app_read` → `app_write` → `app_admin`), grant each tier the *minimum* it needs, wire two login roles to the right tiers, and then **prove** the model with privilege-inspection functions — including a negative test that a read role *cannot* write.

**Estimated time:** ~1.5 hours.

## Setup

In `psql` on your `crunch_w10` scratch database, create a small schema owned by a dedicated migrator role:

```sql
-- The role that owns/creates objects (not an app login)
CREATE ROLE migrator NOLOGIN;
GRANT CREATE, USAGE ON SCHEMA public TO migrator;

-- Two tables the app will use, created AS migrator so ownership is clean
SET ROLE migrator;
CREATE TABLE customers (
  id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name  text NOT NULL,
  email text NOT NULL,
  ssn   text                       -- sensitive: analytics must never read this
);
CREATE TABLE orders (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customers(id),
  total       numeric(10,2) NOT NULL
);
INSERT INTO customers (name, email, ssn) VALUES
  ('Ada',  'ada@example.com',  '111-11-1111'),
  ('Grace','grace@example.com','222-22-2222');
INSERT INTO orders (customer_id, total) VALUES (1, 42.00), (2, 99.50);
RESET ROLE;
```

## Steps

1. **Harden the schema.** Strip the `PUBLIC` pseudo-role so only roles you name can connect/create:
   ```sql
   REVOKE CREATE ON SCHEMA public FROM PUBLIC;
   ```

2. **Create the three group roles** (all `NOLOGIN`): `app_read`, `app_write`, `app_admin`. Make `app_write` a member of `app_read`, and `app_admin` a member of `app_write`, so each tier inherits the one below it.

3. **Grant the read tier.** Give `app_read`: `USAGE` on schema `public`, and `SELECT` on `orders` and on **only the non-sensitive columns** of `customers` (`id, name, email` — *not* `ssn`).

4. **Grant the write tier.** Give `app_write` `INSERT, UPDATE, DELETE` on both tables. (It already reads via `app_read`.)

5. **Grant the admin tier.** Give `app_admin` `TRUNCATE` on both tables in addition to what it inherits.

6. **Handle future tables.** Add an `ALTER DEFAULT PRIVILEGES FOR ROLE migrator` rule so any table `migrator` creates later is automatically `SELECT`-able by `app_read` and writable by `app_write`.

7. **Create two login roles and attach them:**
   ```sql
   CREATE ROLE reporting_svc LOGIN PASSWORD 'demo';
   CREATE ROLE web_app       LOGIN PASSWORD 'demo';
   GRANT CONNECT ON DATABASE crunch_w10 TO reporting_svc, web_app;
   ```
   Grant `app_read` to `reporting_svc` and `app_write` to `web_app`.

8. **Prove it — positive tests.** Using the inspection functions, confirm the intended access:
   ```sql
   SELECT has_table_privilege('web_app',       'orders',    'INSERT') AS web_can_insert;   -- expect t
   SELECT has_table_privilege('reporting_svc', 'orders',    'SELECT') AS rpt_can_read;      -- expect t
   SELECT has_column_privilege('reporting_svc','customers', 'name',  'SELECT') AS name_ok;  -- expect t
   ```

9. **Prove it — negative tests** (this is the important half):
   ```sql
   SELECT has_table_privilege('reporting_svc','orders',    'INSERT') AS rpt_can_write;      -- expect f
   SELECT has_column_privilege('reporting_svc','customers','ssn','SELECT') AS ssn_leak;     -- expect f
   ```

10. **Test from the outside.** In a second terminal, connect as `web_app` and confirm `SELECT ssn FROM customers` errors, while `SELECT id, name FROM customers` works:
    ```bash
    psql "dbname=crunch_w10 user=web_app password=demo host=localhost"
    ```

## Expected result

- `\du` shows `app_write` "Member of {app_read}" and `app_admin` "Member of {app_write}".
- All positive tests return `t`; both negative tests return `f`.
- As `web_app`, `SELECT ssn FROM customers` fails with `permission denied for table customers` (or column-level denial), and `SELECT id, name FROM customers` succeeds.

## Done when…

- [ ] `PUBLIC` no longer has `CREATE` on `public`.
- [ ] The three group roles exist with the correct membership chain.
- [ ] `app_read` can read every column of `customers` **except** `ssn`.
- [ ] An `ALTER DEFAULT PRIVILEGES` rule exists for `migrator` (verify with `\ddp`).
- [ ] Both login roles resolve to the right tier via `has_table_privilege`.
- [ ] The two negative tests return `f`, and the outside connection as `web_app` cannot read `ssn`.

## Stretch

- Add `app_admin` to a human login role with `NOINHERIT`, and practice the `SET ROLE app_admin` / `RESET ROLE` escalation pattern.
- Write one query against `information_schema.role_table_grants` that lists every privilege every role holds on `customers`, sorted by role — the seed of Challenge 2's audit.
