# Week 9 — Challenges

Two open-ended problems. Unlike the exercises, these have **no single right answer** — they're about judgement. You'll defend a design, not just make something run.

Do them after all three lectures and the exercises. Budget ~75 minutes each.

| # | File | The judgement call | Time |
|---|------|--------------------|------|
| 1 | [challenge-01-materialized-view-refresh-strategy.md](./challenge-01-materialized-view-refresh-strategy.md) | *When* and *how* should an expensive reporting mview refresh? | ~75m |
| 2 | [challenge-02-enforce-business-rule-with-trigger.md](./challenge-02-enforce-business-rule-with-trigger.md) | Enforce a cross-row rule a `CHECK` constraint can't express | ~75m |

## What "good" looks like

Both challenges are graded on reasoning, not just a working script:

- You state your assumptions (workload shape, freshness tolerance, write volume).
- You considered at least one alternative and said why you rejected it.
- Your SQL runs, and you tested the edge cases you claim it handles.
- You can articulate the failure mode of your own design.

## Submission

Commit your SQL plus a `DESIGN.md` (½–1 page) per challenge to `c33-week-09/challenge-0N/`. The `DESIGN.md` is where the marks are — the SQL is table stakes.
