# Week 6 — Challenges

Two open-ended problems. Unlike the exercises, these have **no single right answer** — several indexes could work, and part of the challenge is arguing why yours is the best trade-off. Both run against the `shop` seed database.

1. **[Challenge 1 — Cut a slow query with the right index](challenge-01-cut-a-slow-query.md)** — a realistic reporting query is slow; design the index (or indexes) that fixes it, and prove the win. ~75 min.
2. **[Challenge 2 — Diagnose an unused index](challenge-02-diagnose-unused-index.md)** — an index exists but the planner won't touch it. Find every reason and fix what should be fixed. ~60 min.

## How success is judged

- **Evidence over assertion.** Every claim ("this is faster," "the index is unused") is backed by a plan or a catalog query, pasted in.
- **Reasoning.** You can explain *why* your index works, not just that it did — column order, selectivity, sargability, the scan node chosen.
- **Restraint.** The best answer is often the *fewest* indexes, or *dropping* one. Adding five indexes that each help a little is a worse answer than one that nails it.

There is no starter index you must use and no forbidden approach. Bring the mental model from the three lectures.
