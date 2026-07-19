# Week 11 — Challenges

Two open-ended problems. There's no single right answer — the grade is in the *reasoning*, the trade-offs you name, and how you defend your choice against the obvious objections. Write your solution as a short design doc, as if you were proposing it to a team.

1. **[Challenge 1 — Scale a write-heavy workload](challenge-01-scale-a-write-heavy-workload.md)** — an ingest pipeline is drowning the primary. Design a path through partitioning → replication → sharding, in the right order.
2. **[Challenge 2 — Choose SQL vs NoSQL and defend it](challenge-02-choose-sql-vs-nosql.md)** — three real-world cases; pick a data store for each and defend it with CAP, consistency, and access patterns.

## How these are judged

- **Order of operations.** Did you reach for the cheap tools before the expensive ones? Premature sharding loses points; so does ignoring an obvious partitioning win.
- **Honesty about trade-offs.** Every scaling move costs something. Name the cost.
- **Defensibility.** Could you survive a senior engineer poking holes in it? Anticipate the objections and answer them.

Write each as a 1–2 page design doc. Commit to `c33-week-11/challenge-0X/`.
