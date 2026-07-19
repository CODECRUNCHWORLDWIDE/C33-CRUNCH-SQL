# Challenge 2 — Audit a Privilege Model

**Time:** ~2 hours. **Difficulty:** Medium-Hard.

## Scenario

You've joined a team and inherited a Postgres database whose access model grew by accretion — every "just make it work" grant from the last two years is still there, and nobody can say who can do what anymore. Your job: **audit it**, find every place a role has more access than its purpose justifies (an *over-grant*), and deliver a remediation that tightens without breaking the legitimate callers.

Set up the mess to audit:

```sql
-- Roles (reflecting real drift)
CREATE ROLE app_read  NOLOGIN;
CREATE ROLE app_write NOLOGIN;
CREATE ROLE web_app     LOGIN PASSWORD 'x';
CREATE ROLE analytics   LOGIN PASSWORD 'x';   -- meant to be read-only reporting
CREATE ROLE batch_job   LOGIN PASSWORD 'x';   -- nightly ETL into one table
CREATE ROLE intern      LOGIN PASSWORD 'x';   -- temporary, should be gone

CREATE TABLE customer  (id int PRIMARY KEY, name text, ssn text, email text);
CREATE TABLE orders    (id int PRIMARY KEY, customer_id int, total numeric);
CREATE TABLE audit_log (id int PRIMARY KEY, actor text, action text, at timestamptz);

-- The accumulated grants (some are wrong on purpose)
GRANT app_read  TO web_app, analytics, batch_job, intern;
GRANT app_write TO web_app, batch_job, analytics;         -- analytics should NOT write
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_write;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read;   -- includes customer.ssn
GRANT ALL ON audit_log TO app_write;                        -- audit_log should be append-only
ALTER ROLE analytics WITH SUPERUSER;                        -- someone "fixed a bug" this way
GRANT TRUNCATE ON orders TO batch_job;                      -- ETL only needs INSERT
```

## Your task

Produce a **privilege audit** of this database: enumerate who-can-do-what, flag every over-grant with the reason it's excessive and the risk it creates, then ship a remediation script that fixes them — justifying that no legitimate function breaks.

## What counts as an over-grant here

Use the intended purpose of each role (in the comments above) as ground truth. Examples of what to hunt for — there are more than these:

- A reporting role that can **write**.
- **`SUPERUSER`** on anything that isn't a DBA break-glass account.
- A read role that can see **PII** (`customer.ssn`) it has no business reading.
- An "append-only" table that a role can **`UPDATE`/`DELETE`/`TRUNCATE`**.
- A stale role (`intern`) that still has access at all.
- `TRUNCATE`/`DELETE` where only `INSERT` is needed.

## Constraints

- Drive the audit from the **catalog**, not memory: use `has_table_privilege`, `has_column_privilege`, `pg_roles`, `pg_has_role`, and `information_schema.role_table_grants`. The audit must be *reproducible* by running your query, not by eyeballing the setup script.
- Remediation must not remove access a role legitimately needs for its stated purpose.
- Don't just drop everyone to zero — that's not an audit, that's an outage.

## Deliverables

1. `audit.sql` — queries that generate the who-can-do-what report. One query should list every role's effective privilege on every table; another should surface dangerous role attributes (`SUPERUSER`, `BYPASSRLS`, `CREATEROLE`).
2. `FINDINGS.md` — a table of findings, one row per over-grant: **role · object · excess privilege · why it's excessive · risk · fix**. Rank by severity.
3. `remediate.sql` — the `REVOKE`/`ALTER ROLE`/`DROP ROLE` statements that fix each finding, with a one-line comment per statement tying it to a finding.
4. A short **verification** section: re-run the audit query after remediation and show the over-grants are gone while the legitimate grants remain.

## Hints

<details>
<summary>Effective vs direct privileges</summary>

`information_schema.role_table_grants` shows *direct* grants. But `analytics` can write because it's a *member* of `app_write` — an indirect grant. `has_table_privilege('analytics','orders','UPDATE')` returns the *effective* answer (following membership), which is what an attacker actually gets. Audit on effective privilege, then trace *why* to find the grant to cut.
</details>

<details>
<summary>Making a table append-only</summary>

Append-only isn't a single flag. Grant `INSERT` (and maybe `SELECT`) but `REVOKE UPDATE, DELETE, TRUNCATE`. For extra assurance, a `BEFORE UPDATE OR DELETE` trigger that `RAISE EXCEPTION`s belts-and-suspenders the grant.
</details>

## Success criteria

- [ ] `audit.sql` reproduces the full privilege picture from the catalog — no hand-maintained lists.
- [ ] `FINDINGS.md` catches **at least six** distinct over-grants, each with a risk and a fix, ranked by severity.
- [ ] `analytics` ends up read-only with **no `SUPERUSER`**; `batch_job` keeps `INSERT` but loses `TRUNCATE`; `audit_log` becomes append-only; `intern` is removed; `app_read` can no longer read `customer.ssn`.
- [ ] Post-remediation, `web_app` and the legitimate paths still work (show the passing checks).
- [ ] Every `remediate.sql` statement is commented with the finding it addresses.

## Why this matters

Access drifts. Every long-lived database accumulates over-grants, and "who can touch the PII table?" is a question you will be asked — by an auditor, a customer's security team, or an incident. Being the person who can answer it *from the catalog* and remediate without breaking prod is a senior-level skill.
