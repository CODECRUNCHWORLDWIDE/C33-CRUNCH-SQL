# Week 10 — Homework

Six problems, ~5 hours total. Do them in a scratch database (`createdb crunch_w10_hw`). Commit each to your portfolio under `c33-week-10/homework/`.

---

## Problem 1 — Role hierarchy from scratch (45 min)

Design and create a role model for a blog platform with these needs:

- `blog_read` — reads posts and comments.
- `blog_write` — everything read can do, plus insert/update/delete posts and comments.
- `blog_moderator` — everything write can do, plus `TRUNCATE` on a `spam` table.
- Two login roles: `public_site` (read) and `admin_panel` (moderator).

**Acceptance:** `roles.sql` creates the hierarchy with correct membership. A `verify.sql` uses `has_table_privilege` to prove `public_site` can read but not write, and `admin_panel` can truncate `spam`. Both a positive and a negative assertion per role.

---

## Problem 2 — The forgotten future table (30 min)

Grant `blog_read` SELECT on all current tables. Then, as your migrator role, create a *new* table `tags`. Show that `blog_read` **cannot** read it. Fix it with `ALTER DEFAULT PRIVILEGES` so the *next* new table is readable automatically, and prove the fix with one more new table.

**Acceptance:** `future.md` shows the failing read, the `ALTER DEFAULT PRIVILEGES` statement, and the passing read on a table created *after* the rule. One sentence: why did `GRANT ... ON ALL TABLES` not cover `tags`?

---

## Problem 3 — RLS isolation, end to end (60 min)

Create a `note(id, tenant_id, body)` table. Enable **and force** RLS. Write a `FOR ALL` policy isolating by `app.tenant_id` with both `USING` and `WITH CHECK`. Seed two tenants.

**Acceptance:** `rls.sql` plus a `proof.md` that pastes, as a non-owning app role:
1. Unset tenant → zero rows.
2. `SET app.tenant_id='1'` → only tenant 1's rows.
3. A cross-tenant `INSERT` rejected.
4. A cross-tenant `UPDATE ... SET tenant_id=` rejected.

State in one sentence what would change if you *hadn't* run `FORCE ROW LEVEL SECURITY` and the app owned the table.

---

## Problem 4 — PERMISSIVE vs RESTRICTIVE (45 min)

On the `note` table from Problem 3, add a `RESTRICTIVE` policy that forbids reading notes whose `body` starts with `'DRAFT:'` unless a session var `app.show_drafts` is `'on'`.

**Acceptance:** `restrictive.md` demonstrates the same tenant seeing fewer rows with `app.show_drafts` unset than with it `'on'`. In two sentences, explain how the RESTRICTIVE policy combined with the existing PERMISSIVE tenant policy (AND vs OR) to produce that result.

---

## Problem 5 — Constraints, including EXCLUDE (45 min)

Model a `shift(id, employee, during tstzrange)` table where **one employee can't be scheduled for two overlapping shifts**. Add the `EXCLUDE` constraint (remember `btree_gist`). Also add a `CHECK` that `during` is non-empty and forward.

**Acceptance:** `shifts.sql` plus `tests.md` showing: an overlapping shift for the same employee is rejected; an adjacent shift (`[)` touching) is accepted; the same time for a *different* employee is accepted; an empty/backward range is rejected. One sentence: why can't a plain `UNIQUE` express this?

---

## Problem 6 — Kill an injection (60 min)

Here is a vulnerable lookup (choose Python `sqlite3`/psycopg or Node `pg`):

```python
def user_by_login(conn, username, table):
    sql = "SELECT * FROM " + table + " WHERE username = '" + username + "'"
    return conn.execute(sql).fetchall()
```

**Acceptance:** `injection.md` containing:
1. The exact SQL string produced by `username = "admin' --"` and one sentence on what it does.
2. A rewrite that binds `username` as a parameter and allow-lists `table` against a known set (placeholders can't bind identifiers — say why).
3. Proof the payload now returns zero rows / errors safely instead of logging in as admin.
4. One paragraph: "escaping quotes myself would be enough" — argue why that's false.

---

## Time budget

| Problem | Time |
|--------:|----:|
| 1 | 45 min |
| 2 | 30 min |
| 3 | 1 h |
| 4 | 45 min |
| 5 | 45 min |
| 6 | 1 h |
| **Total** | **~5 h** |

After homework, take the [quiz](./quiz.md), then ship the [mini-project](./mini-project/README.md).
