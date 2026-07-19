# Week 10 — Quiz

Thirteen questions. Lectures closed. Aim for 11/13 before Week 11. Mostly multiple-choice, a couple short-answer.

---

**Q1.** In PostgreSQL, the difference between a "user" and a "group" is:

- A) Users are stored in `pg_user`, groups in `pg_group`, and they're distinct object types.
- B) There is no difference — both are **roles**; a "user" is just a role with `LOGIN`.
- C) Groups can log in; users cannot.
- D) Users can own tables; groups cannot.

---

**Q2.** A role has `SELECT` on `sales.orders` but every query fails with "permission denied for schema sales." The missing grant is:

- A) `GRANT SELECT ON sales.orders`
- B) `GRANT USAGE ON SCHEMA sales`
- C) `GRANT CONNECT ON DATABASE`
- D) `GRANT CREATE ON SCHEMA sales`

---

**Q3.** You run `GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read`. Tomorrow a migration adds a new table. `app_read`:

- A) Can read it — `ALL TABLES` is a standing rule.
- B) Cannot read it — `ALL TABLES` only covered tables existing at grant time; use `ALTER DEFAULT PRIVILEGES`.
- C) Can read it only after `ANALYZE`.
- D) Can read it only if it's a superuser.

---

**Q4.** With `INHERIT` (the default), a login role that is a member of a group role automatically gets the group's:

- A) Privileges **and** role attributes (like `CREATEDB`).
- B) Privileges, but **not** attributes like `CREATEDB`/`SUPERUSER`.
- C) Attributes, but not privileges.
- D) Nothing until it runs `SET ROLE`.

---

**Q5.** You `ENABLE ROW LEVEL SECURITY` on a table but define **no** policy. A non-owner querying it sees:

- A) All rows (RLS is opt-in per policy).
- B) Zero rows (RLS is fail-closed).
- C) An error.
- D) Only rows it inserted.

---

**Q6.** The difference between a policy's `USING` and `WITH CHECK`:

- A) `USING` filters rows read/targeted; `WITH CHECK` validates rows being written.
- B) `USING` is for SELECT; `WITH CHECK` is for DELETE.
- C) They're synonyms.
- D) `WITH CHECK` filters reads; `USING` validates writes.

---

**Q7.** Your app owns its tables and connects as the owner. You add RLS policies but they seem to do nothing. The cause and fix:

- A) You forgot `ANALYZE`; run it.
- B) RLS doesn't apply to the table owner by default — add `FORCE ROW LEVEL SECURITY` and/or connect as a non-owning role.
- C) Policies only work on superusers.
- D) You must `VACUUM` the table.

---

**Q8.** Why use `current_setting('app.tenant_id', true)` (two-argument) inside a policy rather than `current_setting('app.tenant_id')`?

- A) It's faster.
- B) The two-arg form returns NULL when unset instead of raising an error, so an unset tenant fails **closed** (zero rows).
- C) The one-arg form doesn't exist.
- D) It caches the value across sessions.

---

**Q9.** Two policies apply to the same command: one `PERMISSIVE`, one `RESTRICTIVE`. A row is visible when:

- A) Either policy passes (OR).
- B) At least one PERMISSIVE passes **AND** every RESTRICTIVE passes.
- C) Both must be PERMISSIVE.
- D) The most recently created policy wins.

---

**Q10.** You need "no two reservations for the same room may overlap in time." The right tool is:

- A) `UNIQUE (room_id, during)`.
- B) A `CHECK` constraint.
- C) An `EXCLUDE USING gist (room_id WITH =, during WITH &&)` constraint.
- D) A `NOT NULL` constraint.

---

**Q11.** Which rule **cannot** be a `CHECK` constraint and needs a trigger (or different design)?

- A) `price >= 0`
- B) `end_date > start_date`
- C) "The order total must equal the sum of its line items in another table."
- D) `discount BETWEEN 0 AND 1`

---

**Q12.** (Short answer.) Given `cur.execute("SELECT * FROM users WHERE name = '" + name + "'")`, write what the query becomes when `name = "' OR '1'='1"`, and state in one sentence why a parameterized query makes the same input harmless.

---

**Q13.** (Short answer.) You must let users sort by a column they choose. Placeholders (`$1`/`%s`) can't bind an identifier. Describe the safe way to build `ORDER BY <user column>`.

---

## Answer key

<details>
<summary>Reveal after attempting</summary>

1. **B** — everything is a role; `LOGIN` is what makes one act like a user. `CREATE USER` is `CREATE ROLE ... LOGIN`.
2. **B** — `USAGE` on the schema is required before any table grant inside it takes effect.
3. **B** — `ALL TABLES` is a one-time grant over then-existing tables. `ALTER DEFAULT PRIVILEGES` covers future tables (keyed to the creating role).
4. **B** — `INHERIT` gives privileges, not attributes; you must `SET ROLE` to use attributes like `CREATEDB`.
5. **B** — RLS is fail-closed: enabled with no permissive policy = zero rows for non-owners.
6. **A** — `USING` = which existing rows are visible/targetable; `WITH CHECK` = whether a written row is allowed to exist.
7. **B** — the owner is exempt from RLS by default; `FORCE ROW LEVEL SECURITY` (and/or not owning the tables as the app) fixes it.
8. **B** — the two-arg form's `missing_ok=true` returns NULL instead of erroring, so an unset tenant yields no rows (fail-closed).
9. **B** — visible iff (some PERMISSIVE passes) AND (all RESTRICTIVE pass). Permissive OR-combine and widen; restrictive AND-combine and narrow.
10. **C** — `EXCLUDE` with `&&` on a range and `=` on the room. `UNIQUE` only catches identical values, not overlaps (needs `btree_gist`).
11. **C** — a `CHECK` is per-row and can't reference other rows/tables. Cross-table aggregates need a trigger (and concurrency care) or a different design.
12. Becomes `SELECT * FROM users WHERE name = '' OR '1'='1'` — always true, returns every row. Parameterized, the whole string `' OR '1'='1` is bound as **data**: the engine searches for a user literally *named* that (finds none) because the value never becomes part of the SQL text.
13. Validate the requested column against an **allow-list** of known-good column names (e.g., a set `{name, price, id}`); only then interpolate it — since it's now provably one of a fixed, safe set, it can't carry injection. `format(... %I ...)` in PL/pgSQL is the server-side equivalent, still used only after allow-listing.

</details>

Scored 11+? On to the mini-project. 8–10: re-read the lecture behind each miss. <8: re-read Lectures 1 and 2 from the top before the mini-project.
