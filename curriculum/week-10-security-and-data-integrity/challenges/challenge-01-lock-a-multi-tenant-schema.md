# Challenge 1 — Lock a Multi-Tenant Schema

**Time:** ~2.5 hours. **Difficulty:** Hard.

## Scenario

A small SaaS runs every customer in one shared Postgres database. Three tables carry a `tenant_id`, and today the application "handles isolation" by remembering to add `WHERE tenant_id = ?` to each query. Last week a reporting endpoint forgot, and one tenant briefly saw another's invoices. Leadership wants isolation moved into the database so a forgotten `WHERE` can never leak data again.

Here's the starting schema — everything is currently wide open:

```sql
CREATE TABLE tenant (
  id   int PRIMARY KEY,
  name text NOT NULL
);
CREATE TABLE project (
  id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id int  NOT NULL REFERENCES tenant(id),
  name      text NOT NULL
);
CREATE TABLE invoice (
  id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id  int  NOT NULL REFERENCES tenant(id),
  project_id bigint NOT NULL REFERENCES project(id),
  amount     numeric(12,2) NOT NULL,
  status     text NOT NULL
);

INSERT INTO tenant VALUES (1,'Acme'), (2,'Globex'), (3,'Initech');
-- seed a handful of projects and invoices per tenant yourself
```

## Your task

Deliver a locked-down version of this schema where the application connects as a **non-owning, least-privilege role** and can only ever touch the current tenant's rows — enforced by the engine, not the app.

## Constraints

- The app connects as a role that does **not** own the tables and is **not** a superuser and does **not** have `BYPASSRLS`.
- Tenant identity comes from a session variable (`app.tenant_id`) set per request — no per-tenant roles, no per-tenant tables.
- `project` and `invoice` must both be isolated. Bonus rigor: an `invoice` must belong to a `project` **of the same tenant** — cross-tenant references are forbidden even for a caller who somehow guesses IDs.
- Unset tenant context must yield **zero rows**, never an error and never all rows.
- Isolation must cover reads *and* writes: no tenant can read, insert, update-into, or delete another tenant's rows.

## Deliverables

1. `lockdown.sql` — the roles, grants, `ENABLE`/`FORCE ROW LEVEL SECURITY`, and policies that secure all three tables.
2. `attack.sql` — an adversarial test script run **as the app role** that *attempts* every cross-tenant operation you can think of and shows each one blocked or empty. At minimum:
   - Read another tenant's invoices (`SELECT ... WHERE tenant_id = <other>`).
   - Insert an invoice with another tenant's `tenant_id`.
   - Update one of your rows to another tenant's `tenant_id`.
   - Delete another tenant's row.
   - Run with `app.tenant_id` unset and confirm zero rows.
3. `RATIONALE.md` — 250–400 words: why RLS over app-side filtering, why `FORCE` matters here, how you prevented cross-tenant `project_id` references, and one attack you considered that your design already stops.

## Hints

<details>
<summary>The owner trap</summary>

If the role that runs your queries also *owns* the tables, RLS quietly does nothing for it. Either create the tables as a separate `migrator`/owner and connect the app as a different role, **or** add `FORCE ROW LEVEL SECURITY` — ideally both. Verify by connecting as the app role and checking a cross-tenant `SELECT` returns nothing.
</details>

<details>
<summary>Same-tenant foreign reference</summary>

A plain FK `invoice.project_id → project.id` doesn't stop an invoice for tenant 1 pointing at a project of tenant 2. One clean fix: add a composite unique key `project (id, tenant_id)` and make `invoice` reference `(project_id, tenant_id)` together — now the FK itself requires the tenants to match. RLS handles visibility; this composite FK handles referential correctness.
</details>

<details>
<summary>Writes, not just reads</summary>

`USING` alone lets a tenant *insert* rows tagged with someone else's `tenant_id`. Add `WITH CHECK` (or a `FOR ALL` policy that carries both) so the written value must equal the session tenant.
</details>

## Success criteria

- [ ] The app role owns nothing, has no `SUPERUSER`/`BYPASSRLS`, and holds only the table privileges it needs.
- [ ] All three tables have RLS `ENABLE`d and `FORCE`d with policies isolating by `app.tenant_id`.
- [ ] Every attack in `attack.sql` is demonstrably blocked or returns zero rows — pasted output proves it.
- [ ] A cross-tenant `invoice.project_id` reference is impossible (composite FK or equivalent).
- [ ] Unset tenant context yields zero rows across all three tables.
- [ ] `RATIONALE.md` defends the design and names an attack it stops.

## Why this matters

This is the exact pattern behind real multi-tenant SaaS on Postgres. The engineers who can do this — and *prove* it with an attack script — are the ones trusted to design the data layer. You'll extend this into the graded mini-project next.
