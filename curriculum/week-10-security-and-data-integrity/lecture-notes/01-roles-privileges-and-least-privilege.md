# Lecture 1 — Roles, Privileges & Least Privilege

> **Duration:** ~2 hours. **Outcome:** You can create login and group roles, grant and revoke privileges at every level (database → schema → table → column), design a role hierarchy that new tables inherit automatically, and articulate least privilege well enough to defend a grant in code review.

Every access-control question in Postgres reduces to two words: **role** and **privilege**. A role is *who is asking*. A privilege is *what that role is allowed to do*. Authentication (proving you are that role) happens at connection time and is mostly `pg_hba.conf`'s job — this lecture is about everything that happens *after* you're connected: **authorization**. Get this layer right and a stolen application password can still only do the handful of things you granted it.

## 1. Why the database must enforce access — not just the app

New engineers assume authorization is the application's job: "the API checks permissions, the database just stores bytes." That works until it doesn't:

- A second service connects directly to the database and skips your API entirely.
- A `SELECT *` in an admin tool dumps columns the API would have hidden.
- A SQL-injection bug (Lecture 3) hands an attacker your app's connection — with all its privileges.

If the database itself grants your web app only `SELECT, INSERT, UPDATE` on three tables, then *no bug in the app can `DROP TABLE` or read the `salaries` table* — the engine refuses. That is the entire argument for **least privilege**: give each role the minimum it needs, so the blast radius of any compromise is small. Authorization enforced in the engine is the floor no application bug can fall through.

## 2. Roles are users *and* groups

Postgres unified "users" and "groups" into one object: the **role**. There is no separate `CREATE USER` concept internally — `CREATE USER` is just `CREATE ROLE ... WITH LOGIN`. A role can:

- **Log in** (has `LOGIN`) — behaves like a traditional user account.
- **Not log in** (`NOLOGIN`, the default) — behaves like a group you grant membership into.
- **Contain other roles** — membership is how a "group" works.

```sql
-- A group role (no login): the bundle of privileges
CREATE ROLE app_read NOLOGIN;

-- A login role (a real account) that will be a MEMBER of the group
CREATE ROLE web_app LOGIN PASSWORD 'change-me-in-prod';

-- Make web_app a member of app_read
GRANT app_read TO web_app;
```

Now anything you grant to `app_read` is automatically usable by `web_app`. Grant `app_read` to five more services and they all get the same baseline. This is the core pattern: **grant privileges to group roles, grant group roles to login roles.** Never scatter table grants directly across a dozen user accounts — you will never be able to audit it.

### Key role attributes

| Attribute | Meaning | Default |
|-----------|---------|---------|
| `LOGIN` / `NOLOGIN` | May this role open a connection? | `NOLOGIN` |
| `SUPERUSER` | Bypasses **all** permission checks. Use almost never. | no |
| `CREATEDB` | May create databases. | no |
| `CREATEROLE` | May create/alter/drop *other* roles. | no |
| `INHERIT` / `NOINHERIT` | Auto-use privileges of roles it's a member of? | `INHERIT` |
| `BYPASSRLS` | Ignores row-level security (Lecture 2). Dangerous. | no |
| `CONNECTION LIMIT n` | Max concurrent connections. | -1 (unlimited) |
| `PASSWORD` / `VALID UNTIL` | Credential + expiry. | none |

Inspect them any time with `\du` in `psql`:

```
                             List of roles
 Role name │                    Attributes                    │ Member of
───────────┼──────────────────────────────────────────────────┼────────────
 app_read  │ Cannot login                                     │ {}
 postgres  │ Superuser, Create role, Create DB, Replication   │ {}
 web_app   │                                                  │ {app_read}
```

## 3. Inheritance vs `SET ROLE`

Membership gives a role access to the group's privileges in one of two ways:

- **`INHERIT` (default):** the member *automatically* uses the group's privileges. `web_app` can read `app_read`'s tables without doing anything special.
- **`NOINHERIT`:** the member must explicitly `SET ROLE app_read;` to "become" the group for the duration of the session. Until it does, it has none of the group's privileges.

`NOINHERIT` is a deliberate safety valve for powerful roles: a human DBA might log in as a low-privilege role and only `SET ROLE admin` when they truly need it — the SQL equivalent of `sudo`. For an unattended service, `INHERIT` is almost always what you want.

```sql
-- Human DBA pattern: low privilege by default, escalate on demand
CREATE ROLE dba_admin NOLOGIN CREATEDB CREATEROLE;
CREATE ROLE alice LOGIN NOINHERIT PASSWORD '...';
GRANT dba_admin TO alice;

-- As alice, day-to-day: no admin powers.
-- When she needs them:
SET ROLE dba_admin;      -- now privileged
-- ... do the dangerous thing ...
RESET ROLE;              -- back to plain alice
```

One subtlety worth memorizing: even with `INHERIT`, a member does **not** inherit the group's *role attributes* like `CREATEDB` or `SUPERUSER` — only its *privileges* (table grants, etc.). To use the group's attributes you must `SET ROLE`.

## 4. The privilege verbs

A **privilege** is permission to run one kind of action against one object. The object types and their verbs:

| Object | Privileges |
|--------|-----------|
| Database | `CONNECT`, `CREATE` (make schemas), `TEMPORARY` |
| Schema | `USAGE` (look inside), `CREATE` (make objects in it) |
| Table | `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `REFERENCES`, `TRIGGER` |
| Column | `SELECT`, `INSERT`, `UPDATE` (a subset of the table's, per-column) |
| Sequence | `USAGE`, `SELECT`, `UPDATE` |
| Function | `EXECUTE` |

Two non-obvious traps for beginners:

1. **`USAGE` on the schema is required before any table grant matters.** A role granted `SELECT` on `sales.orders` but *not* `USAGE` on schema `sales` still can't read it. Always grant schema `USAGE` first.
2. **Inserting into a table with a `SERIAL`/`IDENTITY` column needs `USAGE` on the underlying sequence** (for old `SERIAL` columns). Modern `GENERATED ... AS IDENTITY` columns handle this for you — one more reason to prefer them (you saw them in Week 4).

## 5. `GRANT` and `REVOKE`

The syntax is symmetric:

```sql
GRANT   <privileges> ON <object> TO   <role>;
REVOKE  <privileges> ON <object> FROM <role>;
```

Build the read/write hierarchy from the ground up:

```sql
-- 1. Baseline: let the group connect and see the schema
GRANT CONNECT ON DATABASE shop      TO app_read;
GRANT USAGE   ON SCHEMA   public     TO app_read;

-- 2. Read on all *current* tables
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read;

-- 3. A write group that builds ON TOP of read
CREATE ROLE app_write NOLOGIN;
GRANT app_read TO app_write;                              -- write "is a" read
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_write;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_write;   -- for SERIAL inserts

-- 4. Wire the login roles to the right group
GRANT app_read  TO reporting_svc;   -- read-only analytics service
GRANT app_write TO web_app;         -- the app that mutates data
```

### Column-level grants

You can grant below the table. Say `employees` has a `ssn` column analytics must never see:

```sql
GRANT SELECT (id, name, department, hire_date) ON employees TO app_read;
-- No SELECT on ssn/salary at all: SELECT * now ERRORS for app_read.
```

`app_read` running `SELECT * FROM employees` gets `permission denied for column ssn`. It must name the columns it's allowed. This is a clean way to hide PII without a view.

## 6. The `PUBLIC` role — your first hardening step

There is an implicit role called `PUBLIC` that **every** role is a member of. By default, Postgres grants `PUBLIC`:

- `CONNECT` and `TEMPORARY` on new databases,
- `EXECUTE` on new functions,
- and, historically, `CREATE` + `USAGE` on the `public` schema.

That default is why "anyone who can connect can create tables in `public`." The standard first move on any real database is to strip it:

```sql
REVOKE CREATE ON SCHEMA public FROM PUBLIC;      -- no more junk tables
REVOKE ALL ON DATABASE shop     FROM PUBLIC;      -- explicit CONNECT grants only
```

> Postgres 15+ already removed the automatic `CREATE` on `public` for `PUBLIC` — a good change — but you should still run these on any database you inherit and can't vouch for.

## 7. `ALTER DEFAULT PRIVILEGES` — the grant you forget

Here's the failure everyone hits once. You carefully `GRANT SELECT ON ALL TABLES ... TO app_read`. A week later a migration adds `orders_2026`. `app_read` **cannot read it** — `ALL TABLES` grants only the tables that existed *at grant time*. It is not a standing rule.

The fix is default privileges — a rule applied to *future* objects created by a given role:

```sql
-- For every table the migration role creates from now on, auto-grant to the groups
ALTER DEFAULT PRIVILEGES FOR ROLE migrator IN SCHEMA public
  GRANT SELECT ON TABLES TO app_read;

ALTER DEFAULT PRIVILEGES FOR ROLE migrator IN SCHEMA public
  GRANT INSERT, UPDATE, DELETE ON TABLES TO app_write;

ALTER DEFAULT PRIVILEGES FOR ROLE migrator IN SCHEMA public
  GRANT USAGE, SELECT ON SEQUENCES TO app_write;
```

Note the `FOR ROLE migrator`: default privileges are keyed to *who creates the object*. If your migrations run as `migrator`, set them for `migrator`. Set once, and every future table is locked down correctly — no more "the new table isn't readable" support tickets. Inspect them with `\ddp`.

## 8. A least-privilege checklist

When you provision access for a new service, walk this list:

1. Does it need to **log in**? If it's a group, `NOLOGIN`.
2. What is the **smallest verb set**? A read replica consumer needs `SELECT` only — never grant it `app_write` "just in case."
3. Which **objects**, exactly? Prefer naming tables over `ALL TABLES` when the set is small and stable.
4. Any **columns to hide**? Use column grants or a view for PII.
5. Did you handle **future tables** with `ALTER DEFAULT PRIVILEGES`?
6. Did you **strip `PUBLIC`**?
7. Is anything `SUPERUSER` or `BYPASSRLS` that doesn't need to be? Those two attributes defeat everything in Lecture 2 — audit them ruthlessly.

## 9. Reading the grant state

You can't secure what you can't see. Three tools:

```sql
-- Table-level grants, human-readable
\dp employees

-- Or query the catalog directly
SELECT grantee, privilege_type
FROM information_schema.role_table_grants
WHERE table_name = 'employees';

-- Is role X a member of role Y? (recursively)
SELECT pg_has_role('web_app', 'app_write', 'MEMBER');   -- t / f

-- Can this role do this thing? (great for audits)
SELECT has_table_privilege('app_read', 'employees', 'INSERT');  -- should be f
```

`has_table_privilege`, `has_column_privilege`, `has_schema_privilege`, and `pg_has_role` are the backbone of the privilege audit you'll write in Challenge 2 — they let you *assert* the access model rather than eyeball it.

## 10. Check yourself

- What's the difference between a login role and a group role, and how does one become the other?
- A role has `SELECT` on `sales.orders` but queries fail with "permission denied for schema sales." What did you forget to grant?
- You `GRANT SELECT ON ALL TABLES` today; tomorrow's migration adds a table the app can't read. Why, and what's the fix?
- What does `INHERIT` change, and why might a human admin want `NOINHERIT`?
- Why is `REVOKE ... FROM PUBLIC` a routine first step on an inherited database?
- Which two role attributes silently defeat everything you'll build in Lecture 2?

If those are solid, Lecture 2 pushes authorization down to the *row* level.

## Further reading

- **PostgreSQL — Database Roles:** <https://www.postgresql.org/docs/current/user-manag.html>
- **PostgreSQL — Privileges (`GRANT`):** <https://www.postgresql.org/docs/current/ddl-priv.html>
- **PostgreSQL — `ALTER DEFAULT PRIVILEGES`:** <https://www.postgresql.org/docs/current/sql-alterdefaultprivileges.html>
