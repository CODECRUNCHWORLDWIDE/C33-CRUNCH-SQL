# Week 8 — Challenges

Two open-ended problems. Unlike the exercises, there's **no single right answer** — several query shapes reach a correct result, and part of the point is choosing one and defending it. Both are drawn straight from real analytics work.

| # | Challenge | The real-world job it mirrors | Time |
|---|-----------|-------------------------------|-----:|
| 1 | [challenge-01-cohort-retention-analysis.md](./challenge-01-cohort-retention-analysis.md) | The retention triangle every growth/PM team asks for | ~2h |
| 2 | [challenge-02-sessionize-an-event-stream.md](./challenge-02-sessionize-an-event-stream.md) | Turning a raw event firehose into sessions for product analytics | ~2h |

## Ground rules

- **Pure SQL.** No exporting to Python/pandas. That's the whole point of the week — if you can do it in the database, do it in the database.
- Target **PostgreSQL 16**. If you're on SQLite, the challenge notes what to swap (mostly date math and `INTERVAL`).
- Write it as **one query** (CTEs allowed and encouraged) where you can. If you break it into steps, say why.
- Check your answer by hand against a few rows. Cohort and session queries are easy to get *subtly* wrong (off-by-one months, sessions bleeding across users) and still return plausible-looking numbers.

## How success is judged

Each challenge lists its own rubric. Across both, you're graded on:

- **Correctness** — spot-checkable against the seed data.
- **Clarity** — a teammate can read the CTEs and follow the logic.
- **Robustness** — handles the edge cases named in the challenge (empty cohorts, single-event sessions, users with one event).

## Submission

Commit your `.sql` per challenge to `c33-week-08/challenges/`, plus a short `NOTES.md` explaining your approach and any trade-offs.
