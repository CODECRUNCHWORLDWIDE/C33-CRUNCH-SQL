# Week 3 — Challenges

Two open-ended problems. Unlike the exercises, these have **no single right answer** — several correct queries exist, and part of the challenge is choosing a clean one. Both run against the `crunch_shop` seed database (see `exercises/README.md`).

1. **[Challenge 1 — Layered analytics](challenge-01-layered-analytics.md)** — a multi-stage revenue report that has to handle the awkward edges (customers with no orders, refunds, zero-revenue countries). ~75 min.
2. **[Challenge 2 — Top-N per group](challenge-02-top-n-per-group.md)** — "the top 2 products by revenue *within each category*" — the classic problem that subqueries, CTEs, and (Week 8) window functions all attack differently. ~75 min.

## How these are judged

There's no autograder. Judge your own work against the rubric in each file, and against three questions you should ask of every analytics query you write:

- **Is it correct on the edges?** Zero-order customers, refunds, NULLs, empty groups.
- **Is it readable?** Could a teammate follow it in a month? Named CTEs beat deep nesting.
- **Did you verify it?** Not "it ran" — did you check the numbers against a hand count on this small dataset?

Commit your answers to your portfolio under `c33-week-03/challenges/`, each with a short note on the approach you chose and one you rejected.
