# Week 10 — Resources

Free, public, no signup. PostgreSQL 16 docs are the primary source of truth — read them, don't skim.

## Required reading

- **PostgreSQL — Database Roles** — the whole chapter on roles, attributes, and membership: <https://www.postgresql.org/docs/current/user-manag.html>
  *Why:* every role decision you make this week is grounded here.
- **PostgreSQL — Privileges (`GRANT`/`REVOKE`)** — the authoritative list of object types and their privileges: <https://www.postgresql.org/docs/current/ddl-priv.html>
  *Why:* the definitive answer to "what verb do I need to grant?"
- **PostgreSQL — Row Security Policies** — RLS `ENABLE`/`FORCE`, `USING`/`WITH CHECK`, PERMISSIVE/RESTRICTIVE: <https://www.postgresql.org/docs/current/ddl-rowsecurity.html>
  *Why:* the single most important page for the mini-project.
- **OWASP — SQL Injection Prevention Cheat Sheet** — parameterization, allow-listing, what "safe" actually means: <https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html>
  *Why:* the industry-standard checklist for Lecture 3's second half.

## Reference pages (keep open in tabs)

- **`CREATE POLICY`** — every clause, with examples: <https://www.postgresql.org/docs/current/sql-createpolicy.html>
- **`ALTER DEFAULT PRIVILEGES`** — the grant everyone forgets: <https://www.postgresql.org/docs/current/sql-alterdefaultprivileges.html>
- **Constraints (`CHECK`, FK actions, `EXCLUDE`)** — the integrity toolbox in one page: <https://www.postgresql.org/docs/current/ddl-constraints.html>
- **Range Types & exclusion constraints** — the `&&` overlap operator and `tstzrange`: <https://www.postgresql.org/docs/current/rangetypes.html>
- **`current_setting()` / `set_config()`** — the session-variable functions RLS policies read: <https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADMIN-SET>
- **Privilege inspection functions** (`has_table_privilege`, `pg_has_role`, …) — the backbone of your audit: <https://www.postgresql.org/docs/current/functions-info.html#FUNCTIONS-INFO-ACCESS>

## Deeper dives (high quality, free)

- **Bobby Tables — a guide to preventing SQL injection** — parameterized-query examples in every major language: <https://bobby-tables.com/>
  *Why:* copy-paste-correct safe patterns for whatever driver you use.
- **Crunchy Data — Row Level Security for Tenants in Postgres** — a practical multi-tenant RLS walkthrough: <https://www.crunchydata.com/blog/row-level-security-for-tenants-in-postgres>
  *Why:* mirrors this week's mini-project with production framing.
- **PostgreSQL Wiki — Don't Do This** — anti-patterns, including privilege and injection footguns: <https://wiki.postgresql.org/wiki/Don%27t_Do_This>
  *Why:* short, blunt, saves you from classic mistakes.
- **xkcd 327 — "Exploits of a Mom" ("Little Bobby Tables")** — the one-panel origin of the joke: <https://xkcd.com/327/>
  *Why:* the cultural touchstone; also a perfect 10-second explanation of injection.

## Tools

- **`psql` meta-commands** for security work: `\du` (roles), `\dp` / `\z` (table privileges), `\ddp` (default privileges), `\d+ table` (constraints + RLS). Reference: <https://www.postgresql.org/docs/current/app-psql.html>
- **`btree_gist` extension** — required for `EXCLUDE` constraints that mix `=` and `&&`: `CREATE EXTENSION btree_gist;` Docs: <https://www.postgresql.org/docs/current/btree-gist.html>

## Glossary

| Term | Definition |
|------|------------|
| **Role** | A single object that can be a login "user" and/or a "group." `LOGIN` makes it usable as an account. |
| **Privilege** | Permission to run one action (SELECT, INSERT, USAGE, EXECUTE…) on one object. |
| **Least privilege** | Give each role the minimum access its job requires — so any compromise has a small blast radius. |
| **`PUBLIC`** | The implicit pseudo-role every role belongs to; strip its defaults as a first hardening step. |
| **Inheritance** | With `INHERIT`, a member auto-uses a group's privileges; `NOINHERIT` requires `SET ROLE`. |
| **Default privileges** | A standing rule (`ALTER DEFAULT PRIVILEGES`) that auto-grants on *future* objects a role creates. |
| **RLS** | Row-level security: a per-row filter the engine enforces on every statement against a table. |
| **`USING` / `WITH CHECK`** | Policy clauses: which rows are visible/targetable vs. whether a written row is allowed. |
| **`FORCE ROW LEVEL SECURITY`** | Makes RLS apply even to the table's owner (who is otherwise exempt). |
| **PERMISSIVE / RESTRICTIVE** | Policy types that OR-combine (widen) vs. AND-combine (narrow). |
| **`EXCLUDE` constraint** | Generalized `UNIQUE`: forbids rows that conflict under any operator (e.g., overlapping ranges). |
| **Parameterized query** | SQL sent with placeholders bound to values separately, so input can never become code. |
| **`BYPASSRLS`** | A role attribute that ignores all RLS policies — a master key; audit it ruthlessly. |

---

*Broken link? Open an issue. GPL-3.0.*
