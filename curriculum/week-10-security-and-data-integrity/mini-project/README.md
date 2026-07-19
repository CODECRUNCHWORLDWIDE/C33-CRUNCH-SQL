# Mini-Project — Lock a Multi-Tenant Schema with RLS

> Build the complete, defensible data layer for a small multi-tenant SaaS: a least-privilege role hierarchy, row-level security that isolates every tenant, integrity constraints that hold under any writer, and a parameterized access path — then *prove* it all with an adversarial test suite. This is the week's capstone; it ties Lectures 1–3 into one artifact.

**Estimated time:** 3.5 hours.

This is the mini-project named in the syllabus: **"Lock a multi-tenant schema with RLS."** You are the database engineer for **Crunchboard**, a hypothetical shared-database SaaS where many companies (tenants) store their boards, cards, and members. Your deliverable is the database — schema, security, and a proof it can't leak.

---

## The domain

Model three tenant-scoped tables plus a `tenant` table:

- `tenant(id, name)` — the tenants themselves.
- `board(id, tenant_id, title, is_archived)` — each tenant's boards.
- `card(id, tenant_id, board_id, title, due)` — cards on a board.
- `member(id, tenant_id, email, role)` — people in a tenant; `email` is treated as PII.

Seed at least **3 tenants** with a few boards, cards, and members each so isolation is testable.

---

## Requirements

Your database must satisfy all of the following. Each maps to a lecture.

### Roles & privileges (Lecture 1)
- A non-login `migrator` owns every table. The app never owns its tables.
- A `crunch_read` group (SELECT) and `crunch_write` group (SELECT/INSERT/UPDATE/DELETE), with `crunch_write` a member of `crunch_read`.
- A login role `app` (the application) granted `crunch_write`; a login role `reports` granted only `crunch_read`.
- `member.email` (PII) is **not** readable by `crunch_read` — use a column grant or a view.
- `PUBLIC` stripped of `CREATE` on the schema; future tables covered by `ALTER DEFAULT PRIVILEGES`.

### Row-level security (Lecture 2)
- `board`, `card`, and `member` all have RLS `ENABLE`d **and** `FORCE`d.
- Isolation is driven by `app.tenant_id` (session variable). Reads and writes are both confined to the current tenant (`USING` + `WITH CHECK`).
- Unset tenant context yields zero rows.
- One `RESTRICTIVE` policy that hides `is_archived` boards from the `reports` role but not from `app` (use a second session var like `app.include_archived`).

### Integrity (Lecture 3)
- A `card` cannot reference a `board` of a different tenant (composite FK on `(board_id, tenant_id)`).
- A `CHECK` that `member.role` is one of a fixed set (`'owner','admin','viewer'`).
- An `EXCLUDE` or unique rule that fits the domain (e.g., a tenant can't have two boards with the same `title`), justified in your write-up.

### Safe access (Lecture 3)
- Provide `access.py` (or `.js`) with **one** function that opens a connection, sets `app.tenant_id` via a parameter, and runs a parameterized search over `card.title`. No string-concatenated SQL anywhere; any user-chosen sort column is allow-listed.

---

## Milestones

Work in this order; commit at each.

1. **Schema + seed (45 min).** Tables, composite FK, constraints, seed data. Create everything as `migrator`.
2. **Roles + grants (45 min).** Build the hierarchy, hide `email`, strip `PUBLIC`, set default privileges. Verify with `has_table_privilege`.
3. **RLS (60 min).** Enable + force on all three tenant tables; write the isolation policies and the archived-boards restrictive policy.
4. **Safe access path (30 min).** Write `access.py`/`access.js` with the parameterized, tenant-setting query.
5. **Attack suite + write-up (30 min).** `attack.sql` proving isolation, plus `SECURITY.md`.

---

## Deliverables

A directory `c33-week-10/mini-project/` containing:

1. `schema.sql` — tables, constraints, composite FKs.
2. `security.sql` — roles, grants, default privileges, RLS enable/force, policies.
3. `access.py` (or `access.js`) — the parameterized, tenant-scoped access function.
4. `attack.sql` — adversarial tests, run as `app`, each shown blocked or empty.
5. `SECURITY.md` — 300–500 words: how each layer works and what it defends against, plus one paragraph "if the app had a SQL-injection bug, why can't an attacker read another tenant's cards?" (the defense-in-depth argument from Lecture 3 §7).

---

## Acceptance criteria

- [ ] The `app` role owns nothing and has no `SUPERUSER`/`BYPASSRLS`.
- [ ] As `app` with `app.tenant_id` set, only that tenant's boards/cards/members are visible; unset → zero rows.
- [ ] Cross-tenant read, insert, update-away, and delete are all rejected (proved in `attack.sql`).
- [ ] `reports` cannot read `member.email`, and does not see archived boards; `app` does.
- [ ] A card referencing another tenant's board is impossible.
- [ ] `access.py`/`.js` contains no concatenated SQL; sort input is allow-listed.
- [ ] `SECURITY.md` explains the defense-in-depth story convincingly.

---

## Rubric

| Criterion | Weight | "Great" looks like |
|-----------|------:|--------------------|
| Isolation correctness | 30% | Every cross-tenant op blocked; unset → zero rows; proven, not asserted |
| Least privilege | 20% | Every grant justified; PII hidden; `PUBLIC` stripped; defaults set |
| Integrity | 20% | Composite FK + CHECK + a domain-fit EXCLUDE/UNIQUE, all tested |
| Injection safety | 15% | Zero string-built SQL; identifiers allow-listed |
| Write-up | 15% | The defense-in-depth argument is clear and correct |

---

## Why this matters

This *is* the job — a real multi-tenant SaaS backend, secured the way senior engineers secure it: in the engine, in layers, and provable under attack. Keep this project; it's a portfolio piece and a template you'll copy the next time someone says "put all our customers in one database." It also sets up Week 11, where you take this one secured box and make it many.

---

When done: push, then start [Week 11 — Partitioning, Replication & Scaling](../../week-11-partitioning-replication-and-scaling/).
