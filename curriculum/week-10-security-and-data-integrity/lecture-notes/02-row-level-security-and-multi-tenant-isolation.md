# Lecture 2 — Row-Level Security & Multi-Tenant Isolation

> **Duration:** ~2 hours. **Outcome:** You can turn on row-level security, write `USING` and `WITH CHECK` policies that isolate a multi-tenant table by a session variable, reason about `FORCE ROW LEVEL SECURITY` and PERMISSIVE-vs-RESTRICTIVE combination, and explain why the table owner is the account most likely to leak data.

Lecture 1 controlled *which verbs* a role may run and *which tables* they touch. But a single table often holds rows belonging to many principals — a hundred tenants sharing one `documents` table, or every user's rows in one `orders` table. Table-level grants can't express "this role may `SELECT` this table, **but only the rows where `tenant_id` is theirs`.** That is exactly what **row-level security (RLS)** does: it attaches a `WHERE` clause to the table that the engine enforces on *every* query, automatically, and that no query can remove.

## 1. The problem RLS solves

The naive multi-tenant app filters in the application:

```sql
SELECT * FROM documents WHERE tenant_id = 42;   -- app remembers to add this
```

This is one forgotten `WHERE` away from a catastrophic cross-tenant leak. Every new query, every join, every reporting endpoint, every intern's hotfix must remember the filter. Statistically, someone forgets. The 2018+ era of "tenant B saw tenant A's invoices" bugs is almost always a missing `WHERE tenant_id`.

RLS inverts the responsibility. You declare *once*, on the table, "a row is visible only when its `tenant_id` equals the current request's tenant." The engine then silently appends that predicate to **every** statement — `SELECT`, `UPDATE`, `DELETE`, and (via `WITH CHECK`) `INSERT`. Forgetting is no longer possible, because there is nothing for the query author to remember.

## 2. The mechanism: enable, then policy

RLS is off by default. Two independent switches and one or more policies:

```sql
-- 1. Turn RLS on for the table (no policy yet = deny-all to non-owners)
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- 2. Define what "your rows" means
CREATE POLICY tenant_isolation ON documents
  USING (tenant_id = current_setting('app.tenant_id')::int);
```

Two things to internalize immediately:

- **Enabling RLS with *no* policy denies all rows** to everyone except the table owner and superusers. RLS is fail-closed: absence of a permissive policy means "see nothing." That's the safe default.
- A policy is only a **filter**, not a grant. The role still needs the table-level `SELECT`/`INSERT`/... privilege from Lecture 1. RLS *narrows* what a privileged role can touch; it never *widens* it. Privilege and policy are AND-ed.

## 3. How the request tells the database who it is

RLS policies reference *something that identifies the current caller*. The cleanest mechanism for multi-tenancy is a **session-local custom setting** that your connection layer sets on every checkout from the pool:

```sql
-- Per request, right after grabbing the pooled connection:
SET app.tenant_id = '42';       -- or SELECT set_config('app.tenant_id','42', false);

-- Then all queries in this session automatically see only tenant 42's rows.
SELECT * FROM documents;         -- engine adds: AND tenant_id = 42
```

Read it back inside a policy with `current_setting`. Use the **two-argument** form with `missing_ok = true` so an unset variable doesn't throw — you want it to safely yield "no tenant → no rows," not error:

```sql
CREATE POLICY tenant_isolation ON documents
  USING (tenant_id = current_setting('app.tenant_id', true)::int);
```

If `app.tenant_id` is never set, `current_setting(..., true)` returns `NULL`, the comparison is `NULL` (not true), and the caller sees **zero rows** — fail-closed again. Exactly the behavior you want if a code path forgets to set the tenant.

> Why `SET` and not a literal in the policy? Because one policy definition then serves all tenants. The policy is static; the *value* is per-session. `SET LOCAL` inside a transaction scopes it even more tightly (auto-resets at commit) — ideal with connection pools so a value can't leak to the next borrower of the connection.

## 4. `USING` vs `WITH CHECK` — the two halves of a policy

A policy can carry two expressions, and the distinction is the single most important thing in this lecture:

| Clause | Applies to | Question it answers |
|--------|-----------|---------------------|
| `USING` | Rows being **read or targeted** (SELECT, UPDATE, DELETE) | "Which existing rows may this statement *see/touch*?" |
| `WITH CHECK` | Rows being **written** (INSERT, and the *new* version in UPDATE) | "Is the row this statement is *producing* allowed to exist?" |

Why you need both: `USING` alone stops a tenant from *reading* other tenants' rows, but without `WITH CHECK` a tenant could `INSERT` a row with someone else's `tenant_id`, or `UPDATE` its own row to *hand it to* another tenant. `WITH CHECK` guards the values being written.

```sql
CREATE POLICY tenant_rw ON documents
  USING      (tenant_id = current_setting('app.tenant_id', true)::int)   -- read/target mine
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::int);  -- write only mine
```

Now an `INSERT` that sets `tenant_id = 99` while the session is tenant 42 fails with *"new row violates row-level security policy."* If you omit `WITH CHECK`, Postgres reuses the `USING` expression for writes on `ALL`/`UPDATE`/`INSERT`-capable policies — but being explicit is clearer and lets read and write rules differ (e.g., read-only tenants).

### Per-command policies

You can scope a policy to one command with `FOR`:

```sql
CREATE POLICY tenant_select ON documents FOR SELECT
  USING (tenant_id = current_setting('app.tenant_id', true)::int);

CREATE POLICY tenant_insert ON documents FOR INSERT
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::int);

CREATE POLICY tenant_modify ON documents FOR UPDATE
  USING      (tenant_id = current_setting('app.tenant_id', true)::int)
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::int);
```

Splitting by command lets you, say, allow all tenants to `SELECT` a shared "templates" tenant while restricting writes — expressive when you need it, but a single `FOR ALL` policy covers the common case.

## 5. The owner trap: `FORCE ROW LEVEL SECURITY`

Here is the mistake that turns a "secured" table back into a sieve. **RLS does not apply to the table's owner** by default, nor to superusers or `BYPASSRLS` roles. The rationale: the owner is trusted to manage its own table. But in most application setups, *the role the app connects as also owns the tables* — which means RLS silently does nothing for the very role you were trying to constrain.

Two independent fixes, use both:

1. **Don't let the app own its tables.** Have a separate `migrator`/`owner` role create them; the app connects as a low-privilege `web_app` that is *not* the owner. Then RLS applies to `web_app` normally.
2. **`FORCE ROW LEVEL SECURITY`** — makes policies apply even to the owner:

```sql
ALTER TABLE documents FORCE ROW LEVEL SECURITY;
```

For a table where isolation is a hard security boundary, `FORCE` is not optional — set it, so that even a mistaken connection as the owner is still filtered. Remember the two attributes from Lecture 1: a role with `BYPASSRLS` (or `SUPERUSER`) ignores policies entirely even under `FORCE`. Audit those attributes; they are the master keys.

## 6. Combining policies: PERMISSIVE vs RESTRICTIVE

When a table has multiple policies for the same command, they combine — and *how* they combine depends on their type:

| Type | Combination | Mnemonic |
|------|-------------|----------|
| `PERMISSIVE` (default) | OR-ed together | "any permissive policy that passes lets the row through" |
| `RESTRICTIVE` | AND-ed on top | "every restrictive policy must *also* pass" |

The row is visible iff: **(at least one PERMISSIVE policy passes) AND (every RESTRICTIVE policy passes).** So permissive policies *widen* access (more policies = more allowed rows); restrictive policies *narrow* it (each adds a mandatory condition).

A realistic use: isolate by tenant (permissive) **and** additionally forbid reading rows flagged `archived` unless you're an admin (restrictive):

```sql
-- Baseline tenant isolation (permissive)
CREATE POLICY tenant_isolation ON documents
  USING (tenant_id = current_setting('app.tenant_id', true)::int);

-- Extra mandatory rule: no one sees soft-deleted rows unless admin
CREATE POLICY hide_deleted ON documents AS RESTRICTIVE
  USING (deleted_at IS NULL
         OR current_setting('app.is_admin', true) = 'on');
```

A row now passes only if it's the caller's tenant **and** it isn't soft-deleted (or the caller is admin). Reach for `RESTRICTIVE` when you have a rule that must hold *regardless* of any permissive grant.

## 7. Performance: RLS predicates are just `WHERE` clauses

RLS is not free magic — the policy expression is appended to the query and planned like any predicate. Practical consequences:

- **Index the column your policy filters on.** A policy on `tenant_id` wants an index on (or leading with) `tenant_id`, exactly like the manual `WHERE` would. Everything from Week 6 applies.
- **Keep policy expressions cheap and sargable.** `tenant_id = current_setting(...)::int` is a simple equality — great. A policy that calls a slow function per row will hurt on large scans.
- `current_setting()` is treated as `STABLE`, so it's evaluated once per query, not per row — good.

Check the planner is using your index under RLS with the tools from Week 7:

```sql
SET app.tenant_id = '42';
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM documents WHERE created_at > now() - interval '7 days';
-- You should see the tenant_id predicate folded in and an index scan, not a Seq Scan of every tenant.
```

## 8. A complete, correct multi-tenant table

Putting Lectures 1 and 2 together — this is the pattern the mini-project builds on:

```sql
-- Owner/migrator creates the table; the app role does NOT own it.
CREATE TABLE documents (
  id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id  int  NOT NULL,
  title      text NOT NULL,
  body       text,
  deleted_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON documents (tenant_id);           -- policy-friendly index

ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents FORCE  ROW LEVEL SECURITY; -- apply even to the owner

CREATE POLICY tenant_rw ON documents
  USING      (tenant_id = current_setting('app.tenant_id', true)::int)
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::int);

-- Least-privilege grant from Lecture 1
GRANT SELECT, INSERT, UPDATE, DELETE ON documents TO app_write;
```

At runtime the app does exactly one extra thing per request — `SET app.tenant_id = '<id>'` — and every subsequent statement is confined to that tenant. Forgetting the `SET` yields zero rows, not a leak.

## 9. Check yourself

- What happens the instant you `ENABLE ROW LEVEL SECURITY` with no policy defined?
- Why is `USING` insufficient on its own for a table tenants can write to?
- Your app owns its tables and RLS "isn't working." Name the two fixes.
- `current_setting('app.tenant_id')` throws when the var is unset; `current_setting('app.tenant_id', true)` returns NULL. Which do you want in a policy and why?
- You add a second policy and rows *disappear* that used to show. Was it PERMISSIVE or RESTRICTIVE, and how do policies of each type combine?
- Which role attribute lets a role read every tenant's rows even with `FORCE ROW LEVEL SECURITY` on?

If those land, Lecture 3 secures the *data itself* — constraints and injection defense.

## Further reading

- **PostgreSQL — Row Security Policies:** <https://www.postgresql.org/docs/current/ddl-rowsecurity.html>
- **PostgreSQL — `CREATE POLICY`:** <https://www.postgresql.org/docs/current/sql-createpolicy.html>
- **PostgreSQL — `current_setting()` / `set_config()`:** <https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADMIN-SET>
