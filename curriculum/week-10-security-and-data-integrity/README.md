# Week 10 — Security & Data Integrity

> *A database that trusts its callers is a breach waiting for a date. This week you stop trusting the application layer to enforce "who can see what" and push that decision down into the engine itself — where it can't be bypassed by a bug, a rushed feature, or a forgotten `WHERE tenant_id = ?`.*

Welcome to Week 10 of **C33 · Crunch SQL**. So far you have made the database *correct* (Weeks 4–5), *fast* (Weeks 6–7), and *programmable* (Weeks 8–9). Now you make it *safe*. By the end of this week you can hand a single Postgres database to a hundred tenants and prove — not hope — that tenant A can never read tenant B's rows, even if the application has a SQL-injection hole and hands an attacker a raw query.

The through-line of the week is **defense in depth**: roles decide *who you are*, privileges decide *what verbs you may run*, row-level security decides *which rows those verbs touch*, and constraints decide *what data is even allowed to exist*. Layer them and a single mistake in any one layer is caught by the next.

## Learning objectives

By the end of this week, you will be able to:

- **Create** roles and grant/revoke privileges at the database, schema, table, and column level, and explain **least privilege** and role **inheritance**.
- **Distinguish** a login role from a group role, and design a role hierarchy (`app_read` → `app_write` → `app_admin`) that new tables inherit cleanly via default privileges.
- **Write** row-level security (RLS) policies that isolate a multi-tenant table so each tenant sees only its own rows — using `current_setting()` and session variables set per request.
- **Reason about** `FORCE ROW LEVEL SECURITY`, `BYPASSRLS`, `PERMISSIVE` vs `RESTRICTIVE` policies, and why the table owner is the most dangerous account in the system.
- **Enforce** integrity at scale with `CHECK`, `FOREIGN KEY`, `UNIQUE`, `EXCLUDE` constraints and validation triggers — and know when a constraint beats a trigger.
- **Recognize** an injectable query on sight and rewrite it with **parameterized queries** — never string concatenation — across `psql`, PL/pgSQL, and application drivers.

## Prerequisites

- **C33 Weeks 1–9** completed. You need Week 9's PL/pgSQL and triggers, Week 5's transaction model, and comfortable schema design from Week 4.
- **PostgreSQL 16+** running locally (RLS is a Postgres feature; SQLite has no roles or RLS, so this week is Postgres-first). SQLite appears only in Lecture 3 for the injection-defense material, which is universal.
- A superuser or a role with `CREATEROLE` for the exercises. `psql` on your `PATH`.

Verify: `psql --version` should print `psql (PostgreSQL) 16.x` or newer.

## Topics covered

- Authentication vs authorization — the database's job starts *after* login
- Roles as both users and groups; `LOGIN`, `NOLOGIN`, `INHERIT`, membership, `SET ROLE`
- `GRANT` / `REVOKE` on databases, schemas, tables, columns, sequences, functions
- The `PUBLIC` pseudo-role and why `REVOKE ... FROM PUBLIC` is your first hardening step
- `ALTER DEFAULT PRIVILEGES` so future tables are locked down automatically
- Row-level security: `ENABLE`/`FORCE ROW LEVEL SECURITY`, `CREATE POLICY`, `USING` vs `WITH CHECK`
- Session-variable tenancy: `SET app.tenant_id`, `current_setting('app.tenant_id', true)`
- `PERMISSIVE` vs `RESTRICTIVE` policies and how they combine (OR vs AND)
- `EXCLUDE` constraints (the constraint most engineers have never heard of) for no-overlap rules
- SQL injection: how it works, and parameterized queries / prepared statements as the *only* real fix

## Weekly schedule

The schedule below adds up to approximately **28 hours** — this course's standard weekly load. Treat it as a target, not a contract.

| Day       | Focus                                        | Lectures | Exercises | Challenges | Quiz/Read | Homework | Mini-Project | Daily Total |
|-----------|----------------------------------------------|---------:|----------:|-----------:|----------:|---------:|-------------:|------------:|
| Monday    | Roles, privileges, least privilege           |    2h    |    1.5h   |     0h     |    0.5h   |   1h     |     0h       |     5h      |
| Tuesday   | Row-level security + multi-tenant isolation  |    2h    |    2h     |     0h     |    0.5h   |   1h     |     0h       |     5.5h    |
| Wednesday | Integrity at scale + injection defense       |    2h    |    2h     |     0h     |    0.5h   |   1h     |     0h       |     5.5h    |
| Thursday  | Challenge 1 — lock a multi-tenant schema      |    0h    |    0h     |     2.5h   |    0.5h   |   1h     |     0h       |     4h      |
| Friday    | Challenge 2 — audit a privilege model         |    0h    |    0h     |     2h     |    0.5h   |   1h     |     0h       |     3.5h    |
| Saturday  | Mini-project — RLS multi-tenant lockdown      |    0h    |    0h     |     0h     |    0h     |   0h     |     3.5h     |     3.5h    |
| Sunday    | Quiz + reflection                             |    0h    |    0h     |     0h     |    1h     |   0h     |     0h       |     1h      |
| **Total** |                                              | **6h**   | **5.5h**  | **6.5h**   | **3.5h**  | **5h**   | **7h**       | **28h**     |

## How to navigate this week

Work top to bottom — each piece assumes the ones above it.

| # | File | What's inside | Time |
|--:|------|---------------|-----:|
| 1 | [lecture-notes/01-roles-privileges-and-least-privilege.md](./lecture-notes/01-roles-privileges-and-least-privilege.md) | Roles as users + groups, GRANT/REVOKE, least privilege, inheritance, default privileges | 2h |
| 2 | [lecture-notes/02-row-level-security-and-multi-tenant-isolation.md](./lecture-notes/02-row-level-security-and-multi-tenant-isolation.md) | RLS policies, `USING`/`WITH CHECK`, session-variable tenancy, `FORCE`, PERMISSIVE vs RESTRICTIVE | 2h |
| 3 | [lecture-notes/03-integrity-at-scale-and-sql-injection-defense.md](./lecture-notes/03-integrity-at-scale-and-sql-injection-defense.md) | CHECK/FK/UNIQUE/EXCLUDE constraints, validation triggers, parameterized queries vs string-concat | 2h |
| 4 | [exercises/README.md](./exercises/README.md) | Index of the three exercises | — |
| 5 | [exercises/exercise-01-create-roles-and-grants.md](./exercises/exercise-01-create-roles-and-grants.md) | Build a read/write/admin role hierarchy and grant it correctly | 1.5h |
| 6 | [exercises/exercise-02-write-rls-policies.md](./exercises/exercise-02-write-rls-policies.md) | Turn on RLS and write tenant-isolation policies for a real table | 2h |
| 7 | [exercises/exercise-03-constraints-and-safe-queries.md](./exercises/exercise-03-constraints-and-safe-queries.md) | Add constraints, then find and rewrite an injectable query | 2h |
| 8 | [challenges/README.md](./challenges/README.md) | Index of the two challenges | — |
| 9 | [challenges/challenge-01-lock-a-multi-tenant-schema.md](./challenges/challenge-01-lock-a-multi-tenant-schema.md) | Isolate a shared schema so tenants provably can't see each other | 2.5h |
| 10 | [challenges/challenge-02-audit-a-privilege-model.md](./challenges/challenge-02-audit-a-privilege-model.md) | Audit a messy grant model, find every over-grant, and report it | 2h |
| 11 | [mini-project/README.md](./mini-project/README.md) | Lock a multi-tenant schema end-to-end with RLS + roles + constraints | 3.5h |
| 12 | [homework.md](./homework.md) | Six reinforcement problems (~5h) | 5h |
| 13 | [quiz.md](./quiz.md) | 13 self-check questions + answer key | 1h |
| 14 | [resources.md](./resources.md) | Official docs + hand-picked deep dives | — |

## By the end of this week you can…

- Design and grant a least-privilege role hierarchy that stays correct as the schema grows.
- Prove multi-tenant isolation with RLS instead of trusting every query to remember its `WHERE`.
- Choose the right integrity mechanism — constraint vs trigger — for a given rule, including the underused `EXCLUDE`.
- Spot an injectable query and rewrite it safely, and explain to a teammate why "escaping" is not the fix — parameterization is.

## Up next

[Week 11 — Partitioning, Replication & Scaling](../week-11-partitioning-replication-and-scaling/) — once one box is secure, we make it many boxes.

---

*Part of the Code Crunch Worldwide open curriculum · GPL-3.0 · found an error? Open an issue or PR.*
