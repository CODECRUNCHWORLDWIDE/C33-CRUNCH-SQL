# Week 4 — Exercises

Three guided reps, ~30–50 min each. Do them **in order** — each builds the skill the next assumes. Design first (ER), then reason about form (normalize), then build (DDL). Type the SQL yourself; don't paste.

1. **[Exercise 1 — Draw an ER model](exercise-01-draw-an-er-model.md)** — turn a written domain into an entity-relationship diagram with keys and cardinality, including the junction table an M:N needs.
2. **[Exercise 2 — Normalize a wide table](exercise-02-normalize-a-wide-table.md)** — take one messy wide table and walk it 1NF → 2NF → 3NF, naming the dependency and anomaly at each step.
3. **[Exercise 3 — Write the DDL](exercise-03-write-the-ddl.md)** — translate a model into runnable `CREATE TABLE` statements with the full constraint set, and prove the constraints bite.

## Setup — a database to build in

You need one of these. Both are free.

**PostgreSQL 16 (recommended for this week — it actually enforces types and constraints):**

```bash
# macOS (Homebrew)
brew install postgresql@16 && brew services start postgresql@16
# Debian/Ubuntu
sudo apt install postgresql-16
# then create a scratch database and connect:
createdb c33_week4
psql c33_week4
```

**SQLite (zero-setup fallback):**

```bash
sqlite3 c33_week4.db
-- IMPORTANT: turn on FK enforcement every session
PRAGMA foreign_keys = ON;
-- and use STRICT tables so types are actually enforced:
--   CREATE TABLE t (...) STRICT;
```

Keep a scratch file `week4.sql` open in your editor; paste working statements there as you go so you can re-run the whole build with `psql c33_week4 -f week4.sql`.

## Workflow

- Read the exercise fully before typing.
- Design on paper (or in Mermaid/Excalidraw) *before* writing SQL.
- Run every statement; read the error messages — a rejected `INSERT` is a constraint *working*, and the whole point.
- Save your answers into the file each exercise names, and commit them to your portfolio under `c33-week-04/`.
