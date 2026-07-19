# Week 2 — Challenges

Two open-ended problems. Unlike the exercises, these have **no single right
query** — several join strategies reach the same answer, and part of the work is
choosing one you can defend. Do them after the lectures and exercises.

| # | Challenge | Skill focus | Time |
|--:|-----------|-------------|-----:|
| 1 | [Reconstruct a report](./challenge-01-reconstruct-a-report.md) | multi-table joins, grain, `LEFT` vs `INNER` | 75m |
| 2 | [Find the missing matches](./challenge-02-find-the-missing-matches.md) | anti-joins, `NOT EXISTS`, `EXCEPT` | 60m |

Both run against the `crunch_shop` seed database — load it first via
[exercises/README.md](../exercises/README.md).

**How these are judged:** correctness of the result, but also *readability* and
your *reasoning*. For each challenge, commit your SQL plus a short `NOTES.md`
explaining the join choices you made and one alternative you rejected. A query
that returns the right rows but can't be explained is only half the work.
