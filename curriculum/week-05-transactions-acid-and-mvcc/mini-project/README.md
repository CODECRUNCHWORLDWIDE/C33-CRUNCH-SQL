# Mini-Project — Reproduce and Fix a Concurrency Anomaly

> **Goal:** Take one real concurrency anomaly, reproduce it deterministically in two `psql` sessions, diagnose it precisely, fix it, and prove the fix holds — all documented with a **two-session timeline** a teammate could replay. This is the week made concrete: you don't just *know* the anomalies, you can catch one, explain it, and kill it.

**Estimated time:** ~3 hours. **Deliverable:** a folder `c33-week-05/mini-project/` containing `schema.sql`, `timeline.md`, `fix.sql`, and `report.md`.

## The scenario

You're on the team behind a small **inventory service**. Products have a stock count; orders decrement it. The rule is simple and non-negotiable: **stock must never go below zero** — you cannot sell what you don't have. Yet on busy days, support tickets say customers occasionally ordered the last unit *twice*, and the database shows `stock = -1`.

Your job: reproduce the oversell, prove which anomaly causes it, fix it, and prove the fix.

## Starting schema

Put this in `schema.sql`:

```sql
DROP TABLE IF EXISTS products, orders;
CREATE TABLE products (
    id    int PRIMARY KEY,
    name  text NOT NULL,
    stock int  NOT NULL
);
CREATE TABLE orders (
    id         serial PRIMARY KEY,
    product_id int NOT NULL REFERENCES products(id),
    qty        int NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
);
INSERT INTO products VALUES (1, 'Limited Edition Mug', 1);   -- exactly ONE in stock
```

The buggy "place order" logic your service currently runs (read-check-write):

```
s = SELECT stock FROM products WHERE id = 1;      -- read stock
if s >= qty:                                       -- check
    INSERT INTO orders (product_id, qty) VALUES (1, qty);
    UPDATE products SET stock = stock - qty WHERE id = 1;   -- write
else:
    reject the order
```

## Requirements

Your submission must:

1. **Reproduce the oversell.** With two sessions each ordering the single mug (`qty = 1`), interleave the read-check-write so **both** orders are accepted and `stock` ends at **-1**. Capture the exact interleaving.
2. **Name the anomaly.** State precisely which anomaly this is and why the default isolation level allows it. (Hint: both sessions read `stock = 1` before either wrote — this is a lost update / read-check-write race, and the multi-row version of it is a write-skew-flavored serialization anomaly.)
3. **Fix it — and justify the fix you chose over the alternatives.** Pick one primary fix and implement it in `fix.sql`. Acceptable approaches (you must discuss at least two and justify your pick):
   - **Atomic guarded write:** `UPDATE products SET stock = stock - :qty WHERE id = 1 AND stock >= :qty;` and treat *0 rows affected* as "rejected." No separate read.
   - **Pessimistic lock:** `SELECT stock FROM products WHERE id = 1 FOR UPDATE;` before the check, so the second session blocks.
   - **SERIALIZABLE + retry loop:** run at SERIALIZABLE and retry on `40001`.
   - **A `CHECK (stock >= 0)` constraint** as a belt-and-suspenders backstop (discuss why a constraint alone changes a silent oversell into a loud error but doesn't by itself pick a winner cleanly).
4. **Prove the fix.** Re-run the *same* two-session interleaving against the fixed logic and show that exactly **one** order succeeds, the other is cleanly rejected (or retried to rejection), and `stock` ends at **0**, never negative.
5. **Document with a two-session timeline.** The heart of the deliverable — see format below.

## The two-session timeline (required format)

In `timeline.md`, produce a table for *both* the broken run and the fixed run. Every row is one step; the two session columns show exactly what each typed and what it returned. This is the artifact a teammate replays to confirm your finding.

```
### Broken run — oversell reproduced

| Step | Time | Session A | Session B | products.stock |
|-----:|------|-----------|-----------|:--------------:|
| 1 | t0 | BEGIN; | | 1 |
| 2 | t1 | SELECT stock … → 1 | | 1 |
| 3 | t2 | | BEGIN; | 1 |
| 4 | t3 | | SELECT stock … → 1 | 1 |
| 5 | t4 | INSERT order; UPDATE stock=0; | | 0 (A's view) |
| 6 | t5 | COMMIT; | | 0 |
| 7 | t6 | | INSERT order; UPDATE stock=-1; | -1 |
| 8 | t7 | | COMMIT; | **-1  ← oversold** |
```

Then a second table, **Fixed run**, with the same step structure, showing the second session being blocked or rejected and stock ending at 0.

## Milestones

Work in this order — each depends on the last:

1. **Load the schema** and run one order successfully by hand. Confirm the happy path works. (15 min)
2. **Reproduce the bug** deterministically. You must be able to get `-1` *on demand*, not by luck — that means controlling the interleaving step by step across two prompts. (45 min)
3. **Write the broken timeline** while it's fresh. (20 min)
4. **Implement your chosen fix** in `fix.sql`, and implement at least a sketch of one rejected alternative so you can compare. (45 min)
5. **Prove the fix** with the fixed-run timeline showing stock never goes negative. (30 min)
6. **Write `report.md`** — the narrative (see below). (25 min)

## `report.md` — the write-up

One page, covering:

- **The bug:** what oversold, and the anomaly's precise name.
- **Root cause:** why the default isolation level permitted it — in terms of snapshots / the read-check-write race.
- **The fix:** which approach you chose and *why over the alternatives* (name the cost of each rejected option).
- **Proof:** reference your fixed-run timeline; state the invariant that now holds (`stock >= 0` always) and how the fix enforces it.
- **What you'd add in production:** e.g., the `CHECK (stock >= 0)` backstop, a retry loop, monitoring for rejected orders.

## Rubric

| Criterion | Weight | Full marks means… |
|-----------|:------:|--------------------|
| Deterministic reproduction | 20% | You can produce `stock = -1` on demand, with a step-by-step interleaving — not a lucky race. |
| Correct diagnosis | 20% | The anomaly is named precisely and the root cause explained in snapshot / read-check-write terms. |
| Working fix | 25% | The fixed logic lets exactly one order through; stock never goes negative under the same interleaving. |
| Justification over alternatives | 15% | You considered ≥2 approaches and defended your pick by naming the others' costs. |
| Two-session timeline | 15% | Both broken and fixed runs are documented as replayable step-by-step tables. |
| Report clarity | 5% | A teammate could read `report.md` and understand the bug and fix without you present. |

## Stretch goals

- Add a **second product line** and show that with a `CHECK (stock >= 0)` constraint the oversell becomes a loud `ERROR` even without changing the app logic — then explain why a constraint is a *safety net*, not a *fix* (it turns corruption into an error you still have to handle).
- Reproduce and fix the same bug in **SQLite** and explain why the reproduction is *harder* there (one-writer-at-a-time serializability) — what did you have to do to even get two writers to interleave?
- Wrap the fixed logic in a real **retry loop** in Python (`psycopg`) at SERIALIZABLE and run 100 concurrent orders against a stock of 50; assert exactly 50 succeed.

## Submission

Commit `schema.sql`, `timeline.md`, `fix.sql`, and `report.md` to `c33-week-05/mini-project/`.
