# Challenge 1 — Design a production schema

**Type:** Open-ended design. No single right answer — but there are wrong answers, and there are defensible ones.

**Estimated time:** ~1 hour.

## Scenario

You are the first engineer on **Crunchtix**, a ticketing platform. The product team hands you this description:

> Organizers create *events*. Each event has many *ticket types* (e.g. "General $40", "VIP $120"), each with a fixed quantity. Attendees *purchase* tickets — a purchase can include several tickets across ticket types, is paid in one transaction, and issues one *ticket* (with a unique QR code) per seat. At the door, staff *scan* tickets to check people in; a ticket can only be checked in once. Organizers watch a live dashboard: tickets sold, revenue, and check-ins per event, updating every few seconds during a show.

Expected scale within a year: 50k events, 5k ticket types per big event, 20M tickets issued, 20M scans. Peak load is bursty — a popular on-sale can push thousands of purchases per minute.

## Your latency budgets (design to these)

| Operation | Budget (p95) |
|-----------|-------------:|
| Purchase (write a multi-ticket order atomically) | < 150 ms |
| "My tickets" for one attendee | < 50 ms |
| Door scan / check-in (write, must reject a double check-in) | < 40 ms |
| Organizer live dashboard (sold / revenue / check-ins per event) | < 200 ms |

## Deliverables

Produce a `schema-design.md` containing:

1. **DDL** — `CREATE TABLE` statements with types, primary keys, foreign keys, `NOT NULL`, `UNIQUE`, and `CHECK` constraints. Real, runnable PostgreSQL 16.
2. **Indexes** — every index you would create *and the query it serves*. No speculative indexes.
3. **The four budget queries** — write the actual SQL each budget operation runs, and argue (in plan terms, even without running) why it meets its budget.
4. **Trade-off notes** — at least three explicit decisions where you chose one option over another and why.

## Constraints & things you must address

- **Correctness under concurrency (Week 5):** two people must not buy the last ticket; a ticket must not check in twice. Say how the schema + a transaction enforce this (a `UNIQUE` constraint, `SELECT ... FOR UPDATE`, a check-in timestamp that is `NULL` until scanned, etc.).
- **The dashboard is the hard one.** Counting sold/revenue/check-ins per event over 20M rows every few seconds will not meet 200 ms with a naive aggregate. Decide: covering indexes? A denormalized counter kept by a trigger? A materialized view? Defend your choice against its cost.
- **Types matter (Lecture 2):** money is `numeric`, ids are `bigint`, the QR code is `uuid` or a `text` with a `UNIQUE` index, timestamps are `timestamptz`. Justify anything unusual.
- **No EAV.** If you reach for an "attributes" table, stop and reconsider.

## Hints

<details>
<summary>Where people go wrong on the dashboard</summary>

A live per-event dashboard that re-aggregates 20M scan rows every 3 seconds is the trap. Two defensible paths: (a) maintain `events.tickets_sold`, `events.revenue`, `events.checked_in` as columns updated by triggers on purchase/scan — O(1) reads, at the cost of write-time work and a sync obligation; or (b) a materialized view refreshed on a short interval — simpler, but stale by the refresh window. Either can be right. What is *wrong* is a raw `count(*)` over the scans table on every dashboard poll. Name your choice's cost.
</details>

<details>
<summary>The double-check-in guarantee</summary>

A `checked_in_at timestamptz` column that is `NULL` until scanned, plus an atomic `UPDATE tickets SET checked_in_at = now() WHERE id = $1 AND checked_in_at IS NULL RETURNING id;` — if it returns zero rows, someone already checked in. This is a single-statement guarantee that needs no explicit lock. Cheaper and safer than read-then-write.
</details>

## How success is judged

| Criterion | What "great" looks like |
|-----------|-------------------------|
| Correctness | Constraints actually prevent overselling and double check-in |
| Budget realism | Each of the four queries has a credible path to its budget, argued in plan terms |
| Type discipline | Every column is the narrowest correct type; money is never a float |
| Index intent | Every index is tied to a query; none are speculative |
| Trade-off honesty | The dashboard decision names its cost, not just its benefit |
| No anti-patterns | No EAV, no over-indexing, no `text`-for-everything |

There is no answer key — bring your `schema-design.md` to a peer or mentor and defend it. If two reasonable engineers would design it differently, say which fork you took and why. That defense *is* the deliverable.
