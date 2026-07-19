# Challenge 2 — Choose SQL vs NoSQL and defend it

**Time:** ~90 minutes. **Difficulty:** Medium-Hard. **Deliverable:** `datastore-decisions.md`.

## The premise

"Should we use SQL or NoSQL?" is one of the most common — and most cargo-culted — questions in system design. This challenge makes you answer it *well*: with the access pattern, the consistency requirement, and CAP/PACELC, not with "NoSQL scales better."

For **each** of the three cases below, write a recommendation in `datastore-decisions.md`. A complete answer names: **(a)** the data store (be specific — "PostgreSQL," "DynamoDB," "Cassandra," "Redis," "Neo4j," or a combination), **(b)** the access patterns that drive the choice, **(c)** the consistency model you need and can tolerate, **(d)** where it sits on CAP/PACELC, and **(e)** the strongest counter-argument and why you reject it.

## Case 1 — A bank's ledger

A core banking system recording account balances and transfers. Requirements: a transfer must debit one account and credit another **atomically**; a balance read must never show money that doesn't exist; regulators require an auditable, exactly-correct history. Volume is modest by internet standards (millions of accounts, thousands of transfers/second peak).

**Decide:** SQL or NoSQL? Which system? What isolation/consistency guarantee is non-negotiable here, and what happens if you relax it?

## Case 2 — A social feed's "trending posts" widget

A homepage widget showing the 10 most-liked posts in the last hour, across a 200M-user social network. Requirements: extremely high read volume (every homepage load), globally distributed users, and — critically — **it's completely fine if the count is a few seconds stale or slightly approximate.** A post showing 9,412 likes instead of 9,419 harms nobody.

**Decide:** What store(s)? Why is this the textbook case where eventual consistency is not just acceptable but *preferable*? Where does strong consistency actively hurt you here?

## Case 3 — A fraud-detection graph

A payments company wants to detect fraud rings: "find accounts within 3 hops of this flagged account that share a device fingerprint or shipping address." Requirements: deep relationship traversal over a densely connected graph of accounts, devices, and addresses; queries are exploratory and vary; the graph is large but not internet-scale.

**Decide:** Would you model this in PostgreSQL with recursive CTEs, or reach for a graph database? Defend the boundary — at what point does the relational model stop being the right tool for *this specific* query shape?

## Rules of engagement

- **The default is relational.** For each case, if you pick NoSQL you must justify *deviating* from Postgres with a concrete access pattern or consistency requirement — not "it scales."
- **Polyglot answers are allowed and often correct.** Case 2's real answer might be "Postgres is the source of truth for posts, but the trending widget is served from Redis / a precomputed cache." Say so.
- **No hand-waving on consistency.** "Eventually consistent" is a specific claim with specific consequences. State what a user could observe.
- **Rebut yourself.** For each case, write the single strongest objection to your choice and answer it.

## A worked mini-example (for calibration, not one of the three)

> **Case 0 — a user's session store.** *Recommendation:* Redis (key-value). *Access pattern:* pure get/put by session ID, no joins, short-lived, read on every request. *Consistency:* strong-enough (single logical store); losing a session on a crash just logs the user out — low stakes. *CAP/PACELC:* favors low latency (EL); availability matters more than perfect durability. *Counter-argument:* "Just use a Postgres table." *Rebuttal:* works, but sessions are the highest-QPS, lowest-value data you have — a specialized in-memory KV store is faster and takes load off the primary. This is polyglot persistence done right.

Match that shape for the three real cases.

## How success is judged

| Criterion | What "great" looks like |
|-----------|-------------------------|
| Right tool per case | Case 1 → strong-consistency SQL; Case 2 → cache/eventual; Case 3 → defensible either way, boundary named |
| Consistency literacy | Uses the models precisely; states observable consequences |
| CAP/PACELC use | Applies them correctly, not as buzzwords |
| Self-rebuttal | Each choice survives its strongest objection |
| Anti-cargo-cult | Never justifies NoSQL with "it scales" alone |

## Submission

Commit `datastore-decisions.md` to `c33-week-11/challenge-02/`. One page per case, max.
