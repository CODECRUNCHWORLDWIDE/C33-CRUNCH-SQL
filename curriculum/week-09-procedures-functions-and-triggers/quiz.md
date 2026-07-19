# Week 9 — Quiz

Thirteen questions on views, functions, and triggers. Lectures closed. Aim 11/13 before Week 10.

---

**Q1.** What does a plain `VIEW` store on disk?

- A) The full result set of its query.
- B) Nothing but the query definition.
- C) The query definition plus a cached first result.
- D) A copy of the base tables.

---

**Q2.** Selecting from a view whose query takes 8 seconds live will take approximately:

- A) Milliseconds — views cache results.
- B) 8 seconds — the query runs each time.
- C) 4 seconds — views run at half cost.
- D) It depends on the last `REFRESH`.

---

**Q3.** What are the two requirements for `REFRESH MATERIALIZED VIEW CONCURRENTLY`?

- A) A primary key and a `WITH DATA` clause.
- B) A `UNIQUE` index on the mview, and not being inside a larger transaction block.
- C) A trigger and `pg_cron`.
- D) `SUPERUSER` and an exclusive lock.

---

**Q4.** A plain `REFRESH MATERIALIZED VIEW` (no `CONCURRENTLY`) blocks:

- A) Nothing.
- B) Only writers to the base tables.
- C) All readers of the materialized view, until it finishes.
- D) The whole database.

---

**Q5.** You should reach for `LANGUAGE sql` instead of `LANGUAGE plpgsql` when the function:

- A) Needs a loop.
- B) Needs to catch an exception.
- C) Is essentially just a query with no procedural logic.
- D) Must `COMMIT` partway through.

---

**Q6.** Marking a function that reads from a table as `IMMUTABLE` is:

- A) Correct and fastest.
- B) A bug — the planner may cache a stale result.
- C) Required for it to be used in a view.
- D) The same as `STABLE`.

---

**Q7.** After `SELECT ... INTO v_row FROM t WHERE ...`, how do you tell whether a row was found?

- A) Check `v_row IS NULL`.
- B) Check the `FOUND` boolean.
- C) Check `SQLSTATE`.
- D) You can't; use `EXISTS` first.

---

**Q8.** In a trigger function, which is populated during a `DELETE`?

- A) `NEW` only.
- B) `OLD` only.
- C) Both `NEW` and `OLD`.
- D) Neither.

---

**Q9.** You want to modify the row being written (fill a derived column) so the change is stored. You need:

- A) An `AFTER ROW` trigger returning `NEW`.
- B) A `BEFORE ROW` trigger modifying and returning `NEW`.
- C) A `STATEMENT` trigger.
- D) An `INSTEAD OF` trigger.

---

**Q10.** A `FOR EACH STATEMENT` trigger fires:

- A) Once per affected row.
- B) Once per statement, even if zero rows changed.
- C) Only when at least one row changes.
- D) Once per transaction.

---

**Q11.** For an audit-log trigger recording changes that already happened, the correct timing/level is:

- A) `BEFORE` `FOR EACH STATEMENT`.
- B) `AFTER` `FOR EACH ROW`.
- C) `BEFORE` `FOR EACH ROW`.
- D) `INSTEAD OF` `FOR EACH ROW`.

---

**Q12.** A derived value that depends only on *other columns in the same row* is best implemented with:

- A) A `BEFORE` trigger.
- B) An `AFTER` trigger.
- C) A `GENERATED ALWAYS AS (...) STORED` column.
- D) A materialized view.

---

**Q13.** The one thing a stored **procedure** (`CALL`) can do that a **function** cannot:

- A) Return a value.
- B) Take arguments.
- C) `COMMIT` / `ROLLBACK` mid-body.
- D) Be written in PL/pgSQL.

---

## Answer key

<details>
<summary>Reveal after attempting</summary>

1. **B** — a view stores only its definition; it holds no data.
2. **B** — a view re-runs its query every time; it caches nothing.
3. **B** — a `UNIQUE` index is required (to diff rows), and `CONCURRENTLY` can't run inside a larger transaction block.
4. **C** — a plain refresh takes `ACCESS EXCLUSIVE`, blocking all readers of the mview until done. `CONCURRENTLY` avoids this.
5. **C** — use SQL when there's no procedural logic; it's inlinable and faster. Loops/exceptions/`COMMIT` push you to plpgsql (or a procedure).
6. **B** — `IMMUTABLE` promises the result never changes for given inputs; a table read can change, so the planner may serve a stale cached value. Use `STABLE`.
7. **B** — the `FOUND` special variable is set by `SELECT INTO` (and DML). `IF NOT FOUND THEN ...`.
8. **B** — `OLD` holds the row being deleted; `NEW` is `NULL` on `DELETE`.
9. **B** — only a `BEFORE ROW` trigger can modify `NEW` and have the write store the change. `AFTER` is too late.
10. **B** — statement-level fires once per statement regardless of row count, even zero. `NEW`/`OLD` aren't available.
11. **B** — `AFTER FOR EACH ROW`: you want the change that actually happened, per row.
12. **C** — a same-row derived value is a generated column: simpler, can't be wrong, no trigger. Use a `BEFORE` trigger only when the value depends on other rows/tables.
13. **C** — procedures can `COMMIT`/`ROLLBACK` mid-body; functions always run in the caller's single transaction. (Functions also return values and take args — those aren't the distinction.)

</details>

**Scoring:** 11+ move on. 8–10, re-read the lecture behind each miss. <8, re-read all three lectures — this week compounds into Week 10.
