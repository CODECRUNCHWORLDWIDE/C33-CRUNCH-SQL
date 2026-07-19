# Week 10 — Exercises

Three exercises, ~1.5–2 hours each. Run them **in order** — each seeds a schema the next builds on. Type the SQL yourself; don't paste blindly.

1. **[Exercise 1 — Create roles and grants](exercise-01-create-roles-and-grants.md)** — build a `read → write → admin` role hierarchy, grant it with least privilege, and prove the grants with `has_table_privilege`.
2. **[Exercise 2 — Write RLS policies](exercise-02-write-rls-policies.md)** — enable row-level security on a multi-tenant table and write `USING`/`WITH CHECK` policies so tenants can't see each other.
3. **[Exercise 3 — Constraints and safe queries](exercise-03-constraints-and-safe-queries.md)** — add `CHECK`/`EXCLUDE` constraints, then find an injectable query and rewrite it parameterized.

## Setup — a scratch database

Do all three exercises in a throwaway database so you can drop and restart freely:

```bash
createdb crunch_w10          # or: psql -c 'CREATE DATABASE crunch_w10;'
psql crunch_w10              # you're now in psql, connected to the scratch DB
```

You need a role that can `CREATEROLE` (or a superuser) for Exercise 1. The default `postgres` superuser is fine for a local practice database.

## Suggested workflow

- Keep two terminals open: one running `psql` as the admin/owner, one you can reconnect as the low-privilege app role (`psql "dbname=crunch_w10 user=web_app"`) to *test* the restrictions from the outside.
- After every grant or policy, **verify from the restricted role's side** — security you didn't test isn't security.
- Save your working SQL into `solutions.sql` as you go; you'll reuse pieces in the challenges and mini-project.

The goal isn't to finish fast — it's to reach the point where "lock this table down" is a reflex, not a research task.
