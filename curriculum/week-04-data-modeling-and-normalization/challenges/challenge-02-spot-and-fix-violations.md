# Challenge 2 — Spot and Fix Normalization Violations

**Type:** open-ended audit + repair. Defend every change.

**Estimated time:** 1.5–2 hours.

## The scenario

You inherit a database from a departed contractor. Here is the entire "schema" — one table that a startup has been shoving everything into. It "works" until it doesn't.

```sql
CREATE TABLE projects (
    project_id     INTEGER PRIMARY KEY,
    project_name   TEXT,
    client_name    TEXT,
    client_email   TEXT,
    client_phone   TEXT,
    manager_name   TEXT,
    manager_email  TEXT,
    manager_dept   TEXT,
    dept_budget    NUMERIC,
    team_members   TEXT,          -- e.g. 'Ana:Dev, Bo:Design, Cy:QA'
    tech_stack     TEXT,          -- e.g. 'Postgres, React, Redis'
    status         TEXT,          -- free text: 'active', 'Active', 'in progress', 'done'...
    start_date     TEXT,          -- stored as '2025/03/01', '3-1-25', 'March 1'...
    hours_logged   TEXT           -- '120', '120h', 'about 100'
);
```

Sample rows show the rot: the same client appears with three different phone numbers across three projects; one department's budget is recorded differently on every project it owns; `team_members` and `tech_stack` cram lists into single cells; `status` and `start_date` are inconsistent free text.

## Your task

1. **Audit** (`audit.md`): find and name every problem. For each, state:
   - the specific normalization violation or design flaw (1NF? 2NF? 3NF? type/domain problem?),
   - the functional dependency involved (if applicable),
   - a concrete anomaly it causes (give a one-line example: "change the client's phone and it's now inconsistent across their projects").
   You should find **at least eight** distinct problems. Some are normal-form violations; some are type/constraint failures (dates and hours stored as free text, `status` unbounded).
2. **Redesign** (`fixed.sql`): produce a corrected, normalized (≥3NF) schema in PostgreSQL 16 with proper types and constraints. Break the god-table into the tables it's secretly hiding (projects, clients, managers/employees, departments, and the junction tables for team membership and tech stack).
3. **Migrate** (`migration-notes.md`): describe — in prose, no need to script it fully — how you'd move the existing data into the new schema without losing information. Where would parsing the `team_members` string bite you? What do you do with `start_date = 'March 1'`?

## Constraints on your fix

- Every current column's *information* must still be storable (don't drop data — restructure it).
- `status` becomes a bounded `CHECK` (or enum); pick the canonical value set.
- `start_date` becomes `DATE`; `hours_logged` becomes `NUMERIC`.
- The `team_members` M:N (people ↔ projects, with a *role* on the relationship) and the `tech_stack` M:N (projects ↔ technologies) each get a junction table.
- Client and manager/department facts each live in exactly one place.

## Hints

<details>
<summary>The dependencies hiding in the god-table</summary>

- `client_email → client_name, client_phone` — client facts, repeated per project → their own `clients` table (transitive dependency: `project_id → client → client_phone`).
- `manager_email → manager_name, manager_dept` and `manager_dept → dept_budget` — a *chain* of transitive dependencies → `employees` and `departments` tables.
- `team_members` and `tech_stack` — 1NF violations (repeating groups) *and* hidden M:N relationships.
- The free-text `status`, `start_date`, `hours_logged` aren't normal-form issues — they're **domain/type** failures. Fixing them is part of the job.

</details>

## How success is judged

| Dimension | What "good" looks like |
|-----------|------------------------|
| Diagnosis | ≥8 problems found, each with the right violation name + an anomaly example. |
| Repair | Clean ≥3NF schema; every fact in one place; correct types. |
| Constraints | `status` bounded; FKs with sensible `ON DELETE`; the two M:Ns become junctions with role/attribute placed correctly. |
| Migration realism | You confront the messy-string parsing honestly, not hand-wave it. |

There's no single correct schema — a reviewer should be able to follow *why* each of your tables exists. Defend it in `audit.md`.

## Deliverable

Commit `audit.md`, `fixed.sql`, and `migration-notes.md` to `c33-week-04/challenge-02/`. `fixed.sql` must create cleanly in PostgreSQL 16.
