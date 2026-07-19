# Week 7 — Challenges

Two open-ended problems. Unlike the exercises, these have no single correct answer — there are several valid ways to hit the target, and the judgment is in your reasoning and your evidence. Bring a plan for everything you claim.

Both run against the Week 7 dataset built in [`exercises/README.md`](../exercises/README.md). Load it first.

| # | Challenge | What it tests | Time |
|--:|-----------|---------------|------|
| 1 | [challenge-01-ten-seconds-to-under-100ms.md](./challenge-01-ten-seconds-to-under-100ms.md) | End-to-end tuning: read the plan, find the cause, drive latency to target | ~1.5h |
| 2 | [challenge-02-why-the-plan-changed.md](./challenge-02-why-the-plan-changed.md) | Explain *why* a plan flipped after the data grew — the planner's reasoning | ~1h |

## Ground rules

- Every performance claim must be backed by an `EXPLAIN (ANALYZE, BUFFERS)` plan, before and after.
- Warm the cache: run each query twice, report the second run.
- Change one thing at a time. If you added an index *and* rewrote the query, you can't say which helped.
- Prefer the least invasive fix that hits the target. A new index is heavier than an `ANALYZE`; a schema change is heavier than an index.

## Submission

One markdown write-up per challenge under `c33-week-07/challenges/`, each with the before-plan, the change(s) you made, the after-plan, and a paragraph of reasoning.
