# Week 6 — Exercises

Three guided reps, ~45 min each. All three run against the **`shop`** seed database built in [Lecture 1, §0](../lecture-notes/01-how-a-btree-index-works.md). Build it first if you haven't.

1. **[Exercise 1 — B-tree before/after](exercise-01-btree-before-after.md)** — measure a query with `EXPLAIN (ANALYZE, BUFFERS)`, add a B-tree, measure again, and quantify the win.
2. **[Exercise 2 — GIN for JSONB & full-text](exercise-02-gin-jsonb-and-fts.md)** — index a JSONB payload for containment and a text column for full-text search.
3. **[Exercise 3 — Composite, partial, covering](exercise-03-composite-partial-covering.md)** — get the column order right, cut an index down with a partial `WHERE`, and trigger an index-only scan.

## The one rule for this week

**Never claim an index helped without a before/after `EXPLAIN (ANALYZE, BUFFERS)`.** Paste both plans. The numbers are the point — this is the habit the whole course is building toward.

## Suggested workflow

- Keep a `psql shop` session open and a notes file (`solutions.sql` / `notes.md`) beside it.
- For every index: capture the plan **before**, `CREATE INDEX`, run `ANALYZE`, capture the plan **after**.
- Read plans **bottom-up**. Compare `actual time`, `rows`, and especially **`Buffers:`** (pages touched) — buffer counts are the truest measure of work and don't fluctuate with a warm cache the way wall-clock time does.
- After `CREATE INDEX`, run `ANALYZE <table>;` so the planner's statistics reflect the new index before you test.
