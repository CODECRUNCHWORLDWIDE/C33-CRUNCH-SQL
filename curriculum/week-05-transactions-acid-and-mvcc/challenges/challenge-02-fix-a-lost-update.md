# Challenge 2 — Fix a Lost Update

**Type:** open-ended · **Time:** ~1 hour · **Deliverable:** `writeup.md` + working SQL

A lost update is the concurrency bug you are most likely to write yourself: two sessions read a value, each computes a new one, each writes it back, and one write silently erases the other. Here you'll reproduce it, then fix it **three different ways** and argue which you'd ship.

## Setup

```sql
DROP TABLE IF EXISTS counters;
CREATE TABLE counters (name text PRIMARY KEY, value int NOT NULL, version int NOT NULL DEFAULT 0);
INSERT INTO counters VALUES ('page_views', 100, 0);
```

Imagine application code that increments the counter like this (the naïve read-modify-write):

```
v = SELECT value FROM counters WHERE name='page_views';   -- reads 100
-- application computes v + 1 = 101
UPDATE counters SET value = 101 WHERE name='page_views';   -- writes 101
```

## Part 1 — Reproduce the loss

In two sessions, interleave the naïve pattern so an update is lost:

| Step | Session A | Session B |
|-----:|-----------|-----------|
| 1 | `BEGIN;` | `BEGIN;` |
| 2 | `SELECT value FROM counters WHERE name='page_views';` → 100 | |
| 3 | | `SELECT value FROM counters WHERE name='page_views';` → 100 |
| 4 | `UPDATE counters SET value=101 WHERE name='page_views';` | |
| 5 | `COMMIT;` | |
| 6 | | `UPDATE counters SET value=101 WHERE name='page_views';` |
| 7 | | `COMMIT;` |

Two increments happened; the value is **101**, not 102. One is lost. Confirm and record it. In `writeup.md`, explain exactly *where* the information was destroyed.

## Part 2 — Fix it three ways

Implement **all three** and get each to reliably reach 102 (or higher) under the same interleaving. Reset with `UPDATE counters SET value=100, version=0 WHERE name='page_views';` between attempts.

### Fix 1 — Atomic in-place write

Do the arithmetic *in the database*, not the application:

```sql
UPDATE counters SET value = value + 1 WHERE name='page_views';
```

Explain why this is safe even at READ COMMITTED — what does the `UPDATE` do about the row it's modifying that a `SELECT`-then-`UPDATE` doesn't?

### Fix 2 — Pessimistic lock (`SELECT … FOR UPDATE`)

Keep the read-modify-write shape (you might *need* to, if the new value comes from app logic that can't be expressed in SQL), but lock the row on read so the second session waits:

```sql
BEGIN;
SELECT value FROM counters WHERE name='page_views' FOR UPDATE;   -- others block here
-- app computes new value
UPDATE counters SET value = :new WHERE name='page_views';
COMMIT;
```

Show, with the two-session interleaving, that B now *blocks* at its `FOR UPDATE` until A commits, then reads the fresh 101. Explain the cost of holding that lock.

### Fix 3 — Optimistic concurrency (version column)

Don't lock. Read the `version`, and make the write *conditional* on the version not having changed; if it did, you retry:

```sql
-- read
SELECT value, version FROM counters WHERE name='page_views';   -- value=100, version=0
-- write, guarded by the version you read
UPDATE counters SET value = :new, version = version + 1
WHERE name='page_views' AND version = 0;
-- if this UPDATE reports 0 rows affected, someone beat you — re-read and retry
```

Show the interleaving where B's `UPDATE … AND version=0` affects **0 rows** (because A already bumped version to 1), and describe the retry. Explain when optimistic beats pessimistic.

## Part 3 — Argue

In `writeup.md`, recommend **one** fix for each of these contexts and justify:

1. A hot counter incremented millions of times a day (like the page_views one).
2. A read-modify-write where the new value comes from a slow external API call made *between* the read and the write.
3. A rarely-contended record edited by humans in a web form (classic "someone else saved first").

## Constraints

- All three fixes must actually reach the correct total under the Part 1 interleaving — prove it with pasted output.
- You may not "fix" it by using SERIALIZABLE alone *without* discussing that it, too, needs a retry loop (which makes it a cousin of Fix 3).
- Your recommendations in Part 3 must each name the cost of the *rejected* options, not just praise the chosen one.

## How success is judged

| Criterion | Weak | Strong |
|-----------|------|--------|
| Diagnosis | "It's a race condition" | Pinpoints that both sessions read the same value before either wrote |
| Three fixes work | Only the atomic one works | All three reliably reach 102 under the interleaving, with output shown |
| Understanding | Treats the fixes as interchangeable | Explains *why* atomic write is safe, *what* `FOR UPDATE` blocks, *how* the version guard detects the conflict |
| Judgment (Part 3) | Picks the same fix for everything | Matches fix to context and names the cost of the alternatives |

Full marks: the bug reproduced and explained, three working fixes with output, and three context-appropriate recommendations each defended against its alternatives.

## Deliverable

`writeup.md` plus the SQL for each fix. Commit to `c33-week-05/challenge-02/`.
