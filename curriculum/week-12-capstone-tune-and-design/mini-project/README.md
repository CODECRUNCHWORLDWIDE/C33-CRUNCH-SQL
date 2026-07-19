# Mini-Project (CAPSTONE) — Tune a database to a latency target

> Take a slow, seeded database and drive it to a stated latency target. Deliver a **written tuning report** with before/after `EXPLAIN` plans, the exact changes you made, and *why* each one helped. This is the capstone of C33 — the single most convincing artifact in your portfolio.

**Estimated time:** 7.5 hours, spread across Thursday–Sunday.

This is the whole course in one deliverable. You will profile, read plans, fix queries *and* schema, verify with numbers, and know when to stop — then write it up so a tech lead would trust your work without re-running it. The report matters as much as the tuning: a fix you cannot explain is a fix nobody can maintain.

---

## The database

Build the seeded, deliberately slow **`crunch_capstone`** database below. It is a content platform: users, posts, comments, and reactions — realistic shapes, realistic slowness (few indexes, one wrong type, one N+1-shaped access pattern, one over-broad query).

```bash
createdb crunch_capstone
psql crunch_capstone
```

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

CREATE TABLE users (
    id         bigserial PRIMARY KEY,
    handle     text NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE posts (
    id         bigserial PRIMARY KEY,
    author_id  bigint NOT NULL REFERENCES users(id),
    title      text NOT NULL,
    body       text NOT NULL,
    status     text NOT NULL,               -- 'published','draft','removed'
    created_at timestamptz NOT NULL
);

CREATE TABLE comments (
    id         bigserial PRIMARY KEY,
    post_id    bigint NOT NULL REFERENCES posts(id),
    author_id  bigint NOT NULL REFERENCES users(id),
    body       text NOT NULL,
    created_at timestamptz NOT NULL
);

-- NOTE the wrong type on purpose: reaction "score" stored as text.
CREATE TABLE reactions (
    id         bigserial PRIMARY KEY,
    post_id    bigint NOT NULL REFERENCES posts(id),
    user_id    bigint NOT NULL REFERENCES users(id),
    score      text NOT NULL,               -- '1'..'5' stored as text (bug)
    created_at timestamptz NOT NULL
);

-- 500k users
INSERT INTO users (handle, created_at)
SELECT 'u' || g, now() - (random() * interval '1000 days')
FROM generate_series(1, 500000) g;

-- 3M posts
INSERT INTO posts (author_id, title, body, status, created_at)
SELECT 1 + (random()*499999)::bigint,
       'Post ' || g,
       repeat('lorem ipsum ', 20),
       (ARRAY['published','published','published','draft','removed'])[1+(floor(random()*5))::int],
       now() - (random() * interval '900 days')
FROM generate_series(1, 3000000) g;

-- 12M comments
INSERT INTO comments (post_id, author_id, body, created_at)
SELECT 1 + (random()*2999999)::bigint,
       1 + (random()*499999)::bigint,
       'comment ' || g,
       now() - (random() * interval '800 days')
FROM generate_series(1, 12000000) g;

-- 15M reactions, score as text
INSERT INTO reactions (post_id, user_id, score, created_at)
SELECT 1 + (random()*2999999)::bigint,
       1 + (random()*499999)::bigint,
       (1 + floor(random()*5))::int::text,
       now() - (random() * interval '800 days')
FROM generate_series(1, 15000000) g;

ANALYZE;
```

This seed is larger (30M+ rows) and takes several minutes. If your machine struggles, scale each `generate_series` down by 5× — the *lessons* hold, and note the scale in your report.

---

## The workload and the targets

These five operations run against the platform. Your job: get each under its **p95 budget**.

| # | Operation | Query (shape) | Budget (p95) |
|---|-----------|---------------|-------------:|
| W1 | A user's own posts, newest first | `WHERE author_id = ? AND status='published' ORDER BY created_at DESC LIMIT 20` | < 20 ms |
| W2 | A post's comment thread | `WHERE post_id = ? ORDER BY created_at LIMIT 100` | < 15 ms |
| W3 | Average reaction score for a post | `SELECT avg(score) FROM reactions WHERE post_id = ?` | < 15 ms |
| W4 | "Feed": recent published posts across everyone | `WHERE status='published' ORDER BY created_at DESC LIMIT 50` | < 40 ms |
| W5 | Top posts by comment count in the last 30 days | `join + group by + order by count DESC LIMIT 10` | < 300 ms |

At least one target is impossible without fixing the **schema** (W3's `avg(score)` on a `text` column cannot even run correctly), and at least one needs a rewrite, not just an index. That is the point.

---

## Milestones

### Milestone 1 — Measure (1h)
Enable `pg_stat_statements`, run each workload query many times, and rank by `total_exec_time`. Record the baseline plan and warm timing for all five. **Change nothing yet.**

### Milestone 2 — Schema review (1h)
Review the schema against the Lecture-2 checklist. You should find at least: the `reactions.score` wrong type, unindexed foreign keys, and one query that will need denormalization or a materialized view (W5). Write the review before touching anything.

### Milestone 3 — Fix, one lever at a time (2.5h)
Apply fixes one at a time, re-reading the plan after each:
- Correct `reactions.score` to a real numeric type (a migration — show the `ALTER TABLE ... USING` and its risks).
- Add the indexes each of W1–W4 needs (composite where the query filters *and* sorts).
- Decide W5's approach (covering index, or a rolling materialized view) and justify the cost.
Use `CREATE INDEX CONCURRENTLY`. Make a keep/drop decision on any redundant index.

### Milestone 4 — Verify (1.5h)
For each of the five queries: before/after `EXPLAIN (ANALYZE, BUFFERS)`, warm-run discipline, and a `pgbench` p95. Confirm each is under budget — or document honestly which is not and why.

### Milestone 5 — Write the report (1.5h)
Assemble `tuning-report.md` (structure below). This is the graded artifact.

---

## Deliverable: `tuning-report.md`

A portfolio-quality report with these sections:

1. **Summary** — one paragraph: what the database was, the targets, and the headline result (e.g. "4 of 5 queries under budget; W5 at 340 ms vs. 300 ms target, with a recommended materialized-view path").
2. **Baseline** — the `pg_stat_statements` ranking and a baseline plan + timing per query.
3. **Schema review** — anti-patterns found, with the fix and its trade-off for each.
4. **Changes** — each change as: *hypothesis → the one lever → before/after plan → number → why it worked (in plan terms)*.
5. **Results table** — all five queries, before/after p95 vs. budget, pass/fail.
6. **Costs** — total index size added, write-path impact, and any denormalization's sync burden.
7. **When I stopped** — for each query, the explicit stop decision: under budget, headroom for growth, no longer a top offender.
8. **What I'd do at 10× scale** — a short capacity projection and the next move (partitioning, archiving, replica).

---

## Acceptance criteria

- [ ] `crunch_capstone` seeded (note the scale if you reduced it).
- [ ] Baseline measured with `pg_stat_statements` before any change.
- [ ] The `reactions.score` type bug found and corrected with a real migration.
- [ ] At least 4 of the 5 workload queries provably under their p95 budget; any miss is documented with a path forward.
- [ ] Every change has a **before and after** plan captured with `ANALYZE, BUFFERS`.
- [ ] Changes were applied and measured **one at a time**.
- [ ] A keep/drop decision made on redundant indexes, with reasoning.
- [ ] `tuning-report.md` contains all eight sections.
- [ ] No unexplained "added an index" — every fix has a plan-level *why*.

---

## Rubric

| Criterion | Weight | "Great" looks like |
|-----------|------:|--------------------|
| Measurement first | 15% | Baseline captured before any change; time is spent where it lives |
| Plan literacy | 20% | Every diagnosis names the real bottleneck in plan terms |
| Correct fixes | 20% | Right lever per problem; the type bug is fixed; W5 handled thoughtfully |
| Verification rigor | 20% | Before/after plans + p95, warm-run discipline, honest about misses |
| Schema judgment | 10% | Anti-patterns caught; denormalization (if any) is deliberate and synced |
| Knowing when to stop | 5% | Explicit stop decision per query, tied to the budget |
| Report quality | 10% | A tech lead could trust it without re-running anything |

---

## Why this matters

This report is the artifact that gets you hired as the person who understands the database. It proves you can do the thing the whole industry pretends is mysterious: make a slow database fast, *and explain why*, with evidence. Every downstream course — [C16 Backend](../../../C16-CRUNCH-PRO-WEB-BACKEND/), [C27 Data](../../../C27-CRUNCH-DATA/), [C5 AI & Data Science](../../../C5-CRUNCH-AI-DATA-SCIENCE/) — assumes you can do this. Ship it, put it at the top of your portfolio, and you have finished C33 not as someone who took a SQL course, but as someone who can tune a production database to a number.

---

When done: push `tuning-report.md`, then take the [quiz](../quiz.md) and celebrate — you have completed **C33 · Crunch SQL**.
