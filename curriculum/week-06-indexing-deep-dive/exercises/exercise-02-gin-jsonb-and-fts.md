# Exercise 2 — GIN for JSONB and full-text search

**Goal:** Use a GIN index to accelerate two things a B-tree cannot touch: JSONB containment and full-text search. Compare the two JSONB opclasses and measure each win.

**Estimated time:** 45 minutes.

## Setup

Build the `events` table from [Lecture 2, §3a](../lecture-notes/02-the-other-index-types.md) if you haven't:

```sql
\c shop
CREATE TABLE IF NOT EXISTS events (
    id       bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    payload  jsonb NOT NULL
);
-- (only insert if empty)
INSERT INTO events (payload)
SELECT jsonb_build_object(
    'user_id', 1 + (g % 100000),
    'plan',    (ARRAY['free','pro','team','enterprise'])[1 + (g % 4)],
    'action',  (ARRAY['login','click','purchase','logout'])[1 + (g % 4)],
    'tags',    to_jsonb(ARRAY['t' || (g % 50), 't' || (g % 37)])
)
FROM generate_series(1, 2000000) AS g
WHERE NOT EXISTS (SELECT 1 FROM events);
ANALYZE events;
```

## Part A — JSONB containment, before

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM events WHERE payload @> '{"plan":"enterprise"}';
```

Record the `Seq Scan`, buffers, and time. All 2M rows scanned.

## Part B — Add a GIN index, after

```sql
CREATE INDEX ix_events_payload_gin ON events USING gin (payload);
ANALYZE events;

EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM events WHERE payload @> '{"plan":"enterprise"}';
```

Expect a `Bitmap Index Scan` on the GIN index feeding a `Bitmap Heap Scan`. Record buffers and time; compute the speedup.

## Part C — Compare the two JSONB opclasses

The default `jsonb_ops` indexes keys *and* values and supports `?`. If you only ever use `@>`, `jsonb_path_ops` is smaller and faster. Build both and compare their sizes:

```sql
CREATE INDEX ix_events_pathops ON events USING gin (payload jsonb_path_ops);

SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE relname = 'events'
ORDER BY pg_relation_size(indexrelid) DESC;
```

Record both sizes. In `notes.md`, state which you'd keep for a containment-only workload and what capability you give up.

## Part D — Full-text search with GIN

```sql
-- give customers a searchable bio (from Lecture 2, §3b)
ALTER TABLE customers ADD COLUMN IF NOT EXISTS bio text;
UPDATE customers SET bio =
    'Customer from ' || country || ' who enjoys ' ||
    (ARRAY['hiking','cooking','postgres','music','travel'])[1 + (id % 5)]
WHERE bio IS NULL;

-- before: no FTS index
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM customers
WHERE to_tsvector('english', bio) @@ to_tsquery('english', 'postgres & hiking');

-- add the GIN expression index
CREATE INDEX ix_customers_bio_fts
ON customers USING gin (to_tsvector('english', bio));
ANALYZE customers;

-- after
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM customers
WHERE to_tsvector('english', bio) @@ to_tsquery('english', 'postgres & hiking');
```

Record before/after. Note that the index expression `to_tsvector('english', bio)` must match the query's expression *exactly*, or the index won't be used — try changing `'english'` to `'simple'` in the query only and watch it fall back to a `Seq Scan`.

## Expected result

- JSONB containment: `Seq Scan` (≈2M rows, thousands of buffers) → GIN `Bitmap` scan (hundreds of buffers), a large speedup.
- `jsonb_path_ops` index is noticeably **smaller** than `jsonb_ops`.
- FTS query goes from `Seq Scan` to a GIN-assisted `Bitmap Heap Scan`.

## Done when…

- [ ] `notes.md` has before/after plans for the JSONB query (Part A/B) with buffers and speedup.
- [ ] You recorded both opclass index sizes and stated the trade-off (Part C).
- [ ] `notes.md` has before/after plans for the FTS query (Part D).
- [ ] You demonstrated (or explained) why a mismatched `to_tsvector` config disables the FTS index.

## Stretch

- Index the `tags` array element: `CREATE INDEX ... USING gin ((payload -> 'tags'));` then query `payload -> 'tags' @> '["t5"]'`. Measure.
- Insert 50k new events and time it *with* and *without* the GIN index present. Quantify GIN's write cost (Lecture 2, §3, "the GIN trade-off").
- Add a generated `tsvector` column (`GENERATED ALWAYS AS (to_tsvector('english', bio)) STORED`) and index *that* instead of the expression. Which is cleaner for production, and why?

## Submission

Commit `notes.md` to your portfolio under `c33-week-06/exercise-02/`.
