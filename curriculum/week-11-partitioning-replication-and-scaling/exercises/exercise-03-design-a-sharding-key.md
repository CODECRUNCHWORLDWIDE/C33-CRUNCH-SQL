# Exercise 3 — Design a sharding key

**Goal:** Given a concrete workload, pick a shard key, then stress-test it on paper against the four properties from Lecture 3 §4 — even distribution, query alignment, low cross-shard fan-out, rebalanceability. This is a *design* exercise; you write and defend, you don't stand up a cluster.

**Estimated time:** 60 minutes. **Deliverable:** `sharding-design.md`.

## The workload

You run **CrunchChat**, a team messaging product (think a small Slack). The data model:

- `workspaces` — a customer org. ~50,000 of them. Wildly uneven: the biggest workspace has 40,000 users; the median has 12.
- `channels` — belong to a workspace. ~2M total.
- `messages` — belong to a channel (and thus a workspace). ~50 **billion** and growing by ~200M/day. This is why you're sharding.
- `users` — belong to a workspace.

The hot access patterns, by frequency:

1. **Load a channel's recent messages** (open a channel) — by far the most common read.
2. **Post a message** — the dominant write.
3. **Search messages within a workspace** — common.
4. **Full-text search across ALL workspaces** — rare, admin/analytics only.
5. **Count total messages across the platform** — a daily metrics job.

## Your task

For **each** candidate shard key below, fill in the table in `sharding-design.md`, then pick a winner and defend it.

Candidate keys:

- **A. `message_id` (hash)**
- **B. `channel_id` (hash)**
- **C. `workspace_id` (hash)**
- **D. `created_at` (range)**

For each candidate, score it against the four properties and the five access patterns:

| Candidate | Even distribution? | Query 1 (channel read) | Query 3 (workspace search) | Hot-spot risk | Rebalance cost |
|-----------|--------------------|------------------------|----------------------------|---------------|----------------|
| A message_id | | | | | |
| B channel_id | | | | | |
| C workspace_id | | | | | |
| D created_at | | | | | |

Fill every cell with "single-shard / scatter-gather" for the query columns and a one-line reason for the rest.

## Questions to answer in prose

1. **Which key makes "load a channel's recent messages" a single-shard query?** Explain why that matters given it's the most common read.
2. **`workspace_id` keeps a whole customer together — great for workspace search. What's the catch?** (Reread the "celebrity/whale" problem.) Which specific fact in the workload makes this dangerous?
3. **Why is `created_at` (range) disqualified as the shard key** even though the data is time-series? Where *should* `created_at` show up in this design instead?
4. **Pick a winner.** State the shard key, and explicitly name what you're trading away — which of the five access patterns becomes a scatter-gather, and why that's an acceptable price.
5. **Handle the two rare scatter-gather queries** (4 and 5). How would you serve "search across all workspaces" and "count all messages" *without* making the common path pay for them? (Hint: a separate search index; a rollup/metrics table updated incrementally.)
6. **Global uniqueness.** `message_id` must be unique across all shards, but no shard can see the others. Propose a scheme (UUID? Snowflake-style ID? a central sequence service?) and note its trade-off.

## A hint on the likely answer

There isn't one "correct" key, but a strong design shards messages by **`channel_id`** (hashed): the most common read — open a channel — becomes single-shard, and a channel's messages are naturally co-located. Workspace-search becomes a bounded scatter-gather across only the shards holding that workspace's channels (or you route it to a dedicated search index). Note how this *composes* with Week 1's lecture: within each shard you'd still **partition** the messages table by `created_at` month, so old messages expire cheaply. Sharding across servers, partitioning within them — the two tools stack.

But `workspace_id` is also defensible **if** you special-case the whale workspaces (give the biggest ones their own dedicated shards). The point of this exercise is that you can *argue* your choice, name its failure mode, and have a mitigation — not that you memorized one answer.

## Done when…

- [ ] `sharding-design.md` has the completed 4×5 candidate table.
- [ ] All six prose questions are answered.
- [ ] You committed to one shard key and named exactly what it costs.
- [ ] You proposed a concrete handling for the two rare scatter-gather queries.
- [ ] You have a global-ID scheme with its trade-off stated.

## Stretch

- Sketch what happens when you go from 8 shards to 12. With plain `hash % N`, roughly what fraction of messages must move? Now with **consistent hashing**? Quantify the difference and explain why every large sharded system uses the latter.
- Your biggest workspace grows to be 30% of all traffic on its shard. That shard is now the bottleneck. Describe two ways out (dedicated shard, split by channel) and their downsides.

Commit `sharding-design.md` to `c33-week-11/exercise-03/`.
