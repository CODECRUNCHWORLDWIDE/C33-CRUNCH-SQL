# Challenge 2 — Defend a tuning decision

**Type:** Open-ended argument. The skill under test is not tuning — it is *justifying* a tuning decision with evidence, and knowing when to concede.

**Estimated time:** ~1 hour.

## Scenario

You tuned a slow query on the `crunch_tune` database (or your capstone database) and opened a pull request. A senior reviewer — skeptical by trade, not hostile — leaves comments. Your job is to respond to each with **evidence**, not opinion. Some of their pushback is right; part of the challenge is spotting which.

Pick **one** real change you made this week (an index, a query rewrite, a denormalization, a type fix) and write a `defense.md` that answers all five reviewer comments below about *that specific change*.

## The reviewer's five comments

> **R1.** "How do you know this was actually the bottleneck? Show me it was worth touching at all."

> **R2.** "You added an index. What did it cost? Writes, disk, and does it duplicate an index we already have?"

> **R3.** "Your benchmark ran once and it was fast. Cold cache? p95? How do I know this isn't a lucky warm run?"

> **R4.** "This gets faster now, but what happens at 10× the data? Are you sure the index still helps, or does the whole working set stop fitting in cache?"

> **R5.** "Is there a simpler fix? Could you have solved this without an index — by rewriting the query, fixing a type, or calling it less?"

## What your defense.md must contain

1. **The change**, stated in one line (what you did, to which query/table).
2. **R1 — evidence of impact:** the `pg_stat_statements` row showing this query's share of `total_exec_time` *before* the change. If it was 0.2% of total time, concede that the change did not matter and say so.
3. **R2 — the cost, honestly:** the index's size (`pg_relation_size`), which writes it taxes, and whether it is redundant with an existing index (check `pg_stat_user_indexes` / prefixes). If it *is* redundant, concede and propose dropping one.
4. **R3 — a defensible number:** before/after plans with `BUFFERS`, warm-run discipline, and a p95 from `pgbench` — not a single `EXPLAIN` run.
5. **R4 — the growth argument:** a back-of-envelope projection (rows × width, index growth) and a statement about whether the plan holds at 10×. If the fix is a covering index whose benefit shrinks as the table grows past cache, say so.
6. **R5 — the alternatives you rejected:** name at least one simpler fix you considered (rewrite, type correction, caching, calling it less) and why the one you chose wins — or concede that the simpler fix is better.

## The twist

At least **one** of the reviewer's comments should land — very few real changes survive all five cleanly. A defense that says "you're right, R2 is a redundant index, I'll drop `idx_orders_customer` since `idx_orders_cust_created` covers it" is *stronger*, not weaker, than one that fights every point. Engineering credibility comes from conceding the right things.

## How success is judged

| Criterion | What "great" looks like |
|-----------|-------------------------|
| Evidence over assertion | Every claim is backed by a plan, a number, or a `pg_stat_*` row |
| Honesty | You concede at least one point where the reviewer is right |
| Cost awareness | You name what the change *costs*, not only what it buys |
| Growth thinking | You address 10× data, not just today's numbers |
| Alternatives | You show you considered a simpler fix and can say why yours wins (or doesn't) |

## Stretch

- Swap `defense.md` with a peer and play the reviewer on *their* change. Find the one comment that lands. Reviewing sharpens defending.
- Rewrite one of your defenses as if the reviewer were right on *all five* — what would you have done instead? Sometimes the best defense is a better change.

## Submission

Commit `defense.md` under `c33-week-12/challenge-02/`.
