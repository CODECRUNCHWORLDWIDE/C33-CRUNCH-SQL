# Week 8 — Exercises

Three guided reps. Each is hands-on: you create a small, reproducible dataset, write the query, and check your output against the expected result. Do them in order — each builds on the last.

Everything runs in **PostgreSQL 16** *or* **SQLite 3.25+**. Where the two differ (mostly date arithmetic and `INTERVAL`), the exercise says so. Fire up a scratch database first:

```bash
# PostgreSQL
createdb c33_w8 && psql c33_w8

# or SQLite (a throwaway file DB)
sqlite3 c33_w8.db
```

| # | Exercise | Skills drilled | Time |
|---|----------|----------------|-----:|
| 1 | [exercise-01-ranking-within-partitions.md](./exercise-01-ranking-within-partitions.md) | `PARTITION BY`, `ROW_NUMBER`/`RANK`/`DENSE_RANK`, top-N-per-group, `NTILE` | ~1.5h |
| 2 | [exercise-02-running-totals-and-lag-lead.md](./exercise-02-running-totals-and-lag-lead.md) | Frames, running totals, moving averages, `LAG`/`LEAD`, period-over-period | ~2h |
| 3 | [exercise-03-gaps-and-islands.md](./exercise-03-gaps-and-islands.md) | The row_number-difference trick, `LEAD` gap detection, streaks | ~2h |

## How to work an exercise

1. Paste the **Setup** block into your database.
2. Write your query in a `.sql` file so you can re-run and diff it.
3. Compare against the **Expected result**. If it doesn't match, don't peek at the hint yet — check your `ORDER BY`, your frame, and your tie handling first (that's where 90% of window bugs live).
4. Tick the **Done when…** checklist before moving on.

## Submission

Commit your `.sql` solutions to your portfolio under `c33-week-08/exercises/`. One file per exercise (`exercise-01.sql`, etc.), each with the query and a comment block pasting the output you got.
