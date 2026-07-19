# Exercise 2 — Compare Join Algorithms

**Goal:** Take one join query and force PostgreSQL to run it three ways — nested loop, hash join, merge join — then read the cost and timing of each. You'll *feel* why the planner picks what it picks.

**Estimated time:** 40 minutes.

## Setup

Dataset from [`exercises/README.md`](./README.md) loaded and `ANALYZE`d. Turn on timing:

```sql
\timing on
```

The query under test — all enterprise orders with their customer's city:

```sql
SELECT o.id, o.total, c.city
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE c.tier = 'enterprise';
```

## Steps

### 1. Baseline — let the planner choose

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, o.total, c.city
FROM orders o JOIN customers c ON c.id = o.customer_id
WHERE c.tier = 'enterprise';
```

Record: which join algorithm, total `cost`, and `Execution Time`. This is your reference.

### 2. Force a Hash Join

```sql
SET enable_nestloop = off;
SET enable_mergejoin = off;
-- (hash join is what remains)
EXPLAIN (ANALYZE, BUFFERS) <same query>;
RESET enable_nestloop; RESET enable_mergejoin;
```

Record cost + time. Note the `Hash` node and its `Batches:` count — did the hash fit in `work_mem`?

### 3. Force a Merge Join

```sql
SET enable_nestloop = off;
SET enable_hashjoin = off;
EXPLAIN (ANALYZE, BUFFERS) <same query>;
RESET enable_nestloop; RESET enable_hashjoin;
```

Record cost + time. Note the two `Sort` nodes it had to add — that's the merge join's tax.

### 4. Force a Nested Loop

```sql
SET enable_hashjoin = off;
SET enable_mergejoin = off;
EXPLAIN (ANALYZE, BUFFERS) <same query>;
RESET enable_hashjoin; RESET enable_mergejoin;
```

Record cost + time. Look at the inner Index Scan's `loops` — how many times did it run?

### 5. Fill in the table

In `solutions.md`:

| Algorithm | Total cost | Execution time | Key observation |
|-----------|-----------:|---------------:|-----------------|
| Planner's choice | | | |
| Hash Join | | | |
| Merge Join | | | |
| Nested Loop | | | |

Then answer:

- Which algorithm did the planner pick, and did it match the fastest measured time?
- Why is the Merge Join slower here even though it's "efficient" — what did it have to do first?
- Look at the Nested Loop's inner `loops`. Explain in one sentence why that number makes it slow (or fast) for *this* data size.

## Acceptance criteria

- [ ] All four rows of the table are filled with real numbers from your machine.
- [ ] You reset every `enable_*` flag after each test (verify with `SHOW enable_hashjoin;`).
- [ ] You correctly explain why the planner's pick was or wasn't the fastest.
- [ ] You identified the extra work (sorts) the Merge Join required.

## Stretch

- Rewrite the query to select a *single* enterprise customer's orders (`WHERE c.id = 55`). Re-run the baseline. Does the planner now prefer a Nested Loop? Explain why the shift is correct.
- Raise `work_mem` to `256MB` and re-run the forced Hash Join. Did `Batches:` drop to 1? Did time improve?

## Hints

<details>
<summary>Why forcing works</summary>

`enable_nestloop`, `enable_hashjoin`, `enable_mergejoin` don't truly *disable* a join type — they add a huge cost penalty so the planner avoids it if any alternative exists. That's exactly what you want for a bake-off: it reveals the real runtime of the path the planner rejected, so you can confirm the planner was right (or catch it being wrong). Always `RESET` afterward.

</details>

## Submission

Commit `solutions.md` under `c33-week-07/exercises/exercise-02/`.
