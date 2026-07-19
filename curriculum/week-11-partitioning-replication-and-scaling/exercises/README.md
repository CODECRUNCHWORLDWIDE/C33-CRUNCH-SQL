# Week 11 — Exercises

Three reps, ~1 hour each. Run the commands yourself against a real PostgreSQL 16 instance — don't just read them. Exercise 1 is fully hands-on. Exercise 2 mixes commands you run with concepts you can't fully rehearse on one laptop. Exercise 3 is a design you write and stress-test on paper.

1. **[Exercise 1 — Partition a large table and verify pruning](exercise-01-partition-and-verify-pruning.md)** — build a partitioned table, load millions of rows, and prove pruning fired by reading `EXPLAIN`.
2. **[Exercise 2 — Read-replica setup](exercise-02-read-replica-setup.md)** — stand up a physical read replica (two clusters on one machine), stream WAL, and observe replication lag.
3. **[Exercise 3 — Design a sharding key](exercise-03-design-a-sharding-key.md)** — given a workload, pick a shard key, then stress-test it for hot spots and scatter-gather.

## Suggested workflow

- Have `psql` open in one pane, the exercise in your editor in another.
- Run every command. Read every plan. When a plan surprises you, stop and figure out why before moving on.
- Save your answers, plans, and designs — the mini-project and homework build on them.

The goal isn't to finish fast; it's to build the instinct for *when* each scaling tool earns its complexity.
