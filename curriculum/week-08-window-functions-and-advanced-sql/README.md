# Week 8 — Window Functions & Advanced SQL

> *`GROUP BY` collapses rows. Window functions look across rows **without** collapsing them. The moment that clicks, a whole class of "I'll just do this in Python" problems become one clean query — ranking, running totals, moving averages, gaps-and-islands, sessionization, cohort retention. This week you stop exporting to a spreadsheet.*

Welcome to the analytics core of **C33 · Crunch SQL**. Weeks 1–3 taught you to query, join, and aggregate. Week 8 gives you the one tool that separates people who *use* SQL from people who *reach for pandas the moment a question gets interesting*: the **window function**. By the end of the week you will rank rows inside partitions, compute running totals and moving averages over exactly the frame you intend, look forward and backward with `LAG`/`LEAD`, and assemble the two patterns that show up in every real analytics job — **gaps-and-islands** and **sessionization** — in pure SQL that the database can optimize.

Everything runs on **PostgreSQL 16** (primary) and **SQLite 3.25+** (zero-setup). Window functions are in the SQL standard and both engines support them; the handful of differences are called out where they bite.

## Learning objectives

By the end of this week, you will be able to:

- **Explain** what the `OVER` clause does, and why a window function returns one value *per input row* instead of one value per group.
- **Partition and order** a window with `PARTITION BY` and `ORDER BY`, and predict the output before you run it.
- **Rank** with `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`, `PERCENT_RANK`, and `CUME_DIST` — and know exactly how each treats ties.
- **Frame** a window with `ROWS`, `RANGE`, and `GROUPS`, and state why the *default frame* silently changes your answer when you add `ORDER BY`.
- **Compute** running totals, moving averages, and period-over-period deltas with `SUM(...) OVER`, `AVG(...) OVER`, `LAG`, `LEAD`, `FIRST_VALUE`, and `LAST_VALUE`.
- **Solve** gaps-and-islands (find consecutive runs / detect breaks) using the "row_number difference" trick.
- **Sessionize** an event stream: split a user's activity into sessions by an inactivity gap.
- **Pivot and unpivot** with `CASE`/`FILTER` aggregation, and decide correctly between a window function and a `GROUP BY`.

## Prerequisites

- **Weeks 1–3 of C33** — you can `SELECT`, `JOIN`, `GROUP BY`/`HAVING`, and write a CTE without looking it up. Window functions build directly on aggregation; if `GROUP BY` is shaky, re-read Week 3 first.
- A working `psql` against a local PostgreSQL 16, **or** the `sqlite3` CLI. Both are free; setup is in `resources.md`.
- Comfort reading a small result set and checking it by hand — this week rewards paranoia about ties and frame boundaries.

## Topics covered

- The `OVER` clause: `PARTITION BY`, `ORDER BY`, and the window frame
- Ranking family: `ROW_NUMBER` vs `RANK` vs `DENSE_RANK` vs `NTILE`, plus `PERCENT_RANK`/`CUME_DIST`
- Window frames: `ROWS` vs `RANGE` vs `GROUPS`, `UNBOUNDED PRECEDING`, `CURRENT ROW`, `n FOLLOWING`, `EXCLUDE`
- Running totals, cumulative sums, and moving averages
- Offset functions: `LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`
- The `WINDOW` clause for naming and reusing a window definition
- Gaps-and-islands and event-stream sessionization
- Conditional aggregation: pivot/unpivot with `FILTER` and `CASE`
- Window function vs `GROUP BY` — a decision, not a coin flip

## Weekly schedule

The schedule below adds up to approximately **28 hours** (this course's full-time pace). Treat it as a target.

| Day       | Focus                                          | Lectures | Exercises | Challenges | Quiz/Read | Homework | Mini-Project | Daily Total |
|-----------|------------------------------------------------|---------:|----------:|-----------:|----------:|---------:|-------------:|------------:|
| Monday    | `OVER`, `PARTITION BY`, the ranking family      |    2h    |    1.5h   |     0h     |    0.5h   |   1h     |     0h       |     5h      |
| Tuesday   | Frames, running totals, `LAG`/`LEAD`            |    2h    |    2h     |     0h     |    0.5h   |   1h     |     0h       |     5.5h    |
| Wednesday | Advanced patterns: gaps-and-islands, pivot      |    2h    |    2h     |     1h     |    0.5h   |   1h     |     0h       |     6.5h    |
| Thursday  | Cohort/retention challenge                       |    0h    |    0h     |     2h     |    0.5h   |   1h     |     1.5h     |     5h      |
| Friday    | Sessionization challenge + mini-project start    |    0h    |    0h     |     1h     |    0.5h   |   0h     |     2.5h     |     4h      |
| Saturday  | Mini-project (cohort + running-total dashboard)  |    0h    |    0h     |     0h     |    0h     |   0h     |     1.5h     |     1.5h    |
| Sunday    | Quiz + reflection                                |    0h    |    0h     |     0h     |    0.5h   |   0h     |     0h       |     0.5h    |
| **Total** |                                                | **6h**   | **5.5h**  | **4h**     | **3h**    | **4h**   | **7h**       | **28h**     |

## How to navigate this week

Work top to bottom. Lectures teach the *why*; exercises drill the mechanics; challenges have no single right answer; the mini-project ties it all together.

| # | File | What's inside | Time |
|---|------|---------------|-----:|
| 1 | [lecture-notes/01-window-functions-and-ranking.md](./lecture-notes/01-window-functions-and-ranking.md) | The `OVER` clause, `PARTITION BY`, and the full ranking family | ~2h |
| 2 | [lecture-notes/02-frames-running-totals-and-offsets.md](./lecture-notes/02-frames-running-totals-and-offsets.md) | Window frames, running totals, moving averages, `LAG`/`LEAD` | ~2h |
| 3 | [lecture-notes/03-advanced-patterns-gaps-islands-sessionization-pivot.md](./lecture-notes/03-advanced-patterns-gaps-islands-sessionization-pivot.md) | Gaps-and-islands, sessionization, pivot/unpivot, window vs `GROUP BY` | ~2h |
| 4 | [exercises/README.md](./exercises/README.md) | Index of the three guided reps | — |
| 5 | [exercises/exercise-01-ranking-within-partitions.md](./exercises/exercise-01-ranking-within-partitions.md) | Rank products per category; top-N-per-group | ~1.5h |
| 6 | [exercises/exercise-02-running-totals-and-lag-lead.md](./exercises/exercise-02-running-totals-and-lag-lead.md) | Running totals, moving averages, period-over-period growth | ~2h |
| 7 | [exercises/exercise-03-gaps-and-islands.md](./exercises/exercise-03-gaps-and-islands.md) | Find consecutive runs and missing days | ~2h |
| 8 | [challenges/README.md](./challenges/README.md) | Index of the two open-ended challenges | — |
| 9 | [challenges/challenge-01-cohort-retention-analysis.md](./challenges/challenge-01-cohort-retention-analysis.md) | Build a monthly signup-cohort retention matrix | ~2h |
| 10 | [challenges/challenge-02-sessionize-an-event-stream.md](./challenges/challenge-02-sessionize-an-event-stream.md) | Split raw events into sessions by inactivity gap | ~2h |
| 11 | [mini-project/README.md](./mini-project/README.md) | Cohort + running-total dashboard query (the week's build) | ~7h |
| 12 | [homework.md](./homework.md) | Six practice problems (~4h) | ~4h |
| 13 | [quiz.md](./quiz.md) | 12 self-check questions + answer key | ~0.5h |
| 14 | [resources.md](./resources.md) | Official docs + curated deep dives | — |

## By the end of this week you can…

- Look at a request like *"rank each customer's orders by value and flag their top three"* and write it as one query with no subquery gymnastics.
- Produce a running total and a 7-day moving average side by side in a single `SELECT`.
- Detect gaps in a sequence of dates or IDs, and collapse consecutive rows into runs.
- Turn a firehose of raw events into clean per-user sessions.
- Build a cohort-retention triangle — the chart every growth team asks for — without leaving SQL.
- Explain, in one sentence each, when a window function is the right tool and when a plain `GROUP BY` is cleaner.

## Up next

[Week 9 — Procedures, Functions & Triggers](../week-09-procedures-functions-and-triggers/) — now that you can express analytics in SQL, you'll package logic *inside* the database with PL/pgSQL, views, and triggers.

---

*Part of the Code Crunch Worldwide open curriculum · GPL-3.0. Found an error? Open an issue or PR.*
