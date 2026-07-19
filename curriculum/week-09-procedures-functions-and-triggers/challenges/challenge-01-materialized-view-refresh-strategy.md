# Challenge 1 — Build a Materialized-View Refresh Strategy

**Time:** ~75 minutes. **Difficulty:** Medium. **No single right answer.**

## Scenario

You run the analytics for a mid-size store. The `orders` and `order_items` tables together hold ~40 million rows and grow by ~50,000 rows/day. The business-facing dashboard shows a **daily sales summary**: per day, per country — order count, gross revenue, and average order value.

Computed live, that query aggregates tens of millions of rows and takes ~9 seconds. The dashboard is hit hundreds of times an hour during business hours. Nine seconds per hit is unacceptable; the raw data changing constantly is real.

Your job: **materialize the summary and design a refresh strategy** that keeps the dashboard fast without lying too much about freshness or wrecking write throughput.

## Constraints

- The dashboard must respond in **well under 1 second**.
- Data may be **up to 15 minutes stale** — the business signed off on that.
- Refreshing must **not block** dashboard reads.
- Writes to `orders` (checkout traffic) must **not** be slowed by your refresh mechanism.
- Free tools only. `pg_cron` is available.

## Your deliverables

1. **The materialized view.** Write `CREATE MATERIALIZED VIEW daily_sales_summary AS ...` over `orders` + `order_items` (group by day and country; compute count, revenue, avg order value). Assume `orders(order_id, customer_id, country, status, placed_at)` and `order_items(order_id, quantity, unit_price_cents)`.

2. **The unique index** that makes concurrent refresh possible — and a one-line explanation of *why* it's required.

3. **The refresh mechanism.** Pick one and justify it in `DESIGN.md`:
   - Scheduled (`pg_cron` every N minutes)
   - On-demand (app calls refresh after known bulk events)
   - Trigger-driven (refresh on every base-table write)

4. **The schedule.** If scheduled, what interval, and why that interval given the 15-minute tolerance?

## Hints

<details>
<summary>Which refresh mode blocks readers?</summary>

A plain `REFRESH MATERIALIZED VIEW` takes an `ACCESS EXCLUSIVE` lock — every reader waits. `REFRESH MATERIALIZED VIEW CONCURRENTLY` does not block readers, but requires a `UNIQUE` index on the mview and is somewhat slower. Given "must not block reads," one of these is disqualified.
</details>

<details>
<summary>Why trigger-driven is a trap here</summary>

A trigger that refreshes the mview on every `INSERT INTO orders` would re-aggregate 40M rows on *every checkout* — catastrophic for write latency and completely redundant (50,000 refreshes/day for data allowed to be 15 minutes old). Map each constraint to each strategy before you choose.
</details>

<details>
<summary>pg_cron skeleton</summary>

```sql
SELECT cron.schedule(
    'refresh-daily-sales',
    '*/10 * * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales_summary'
);
```
Pick the cron expression that fits your chosen interval.
</details>

<details>
<summary>Testing without 40M rows</summary>

You don't need real scale. Build the schema, insert a few thousand rows with `generate_series`, create the mview + unique index, and prove that `REFRESH ... CONCURRENTLY` succeeds and that a second session can `SELECT` from the mview *during* a refresh (open two `psql` sessions).
</details>

## How success is judged

| Criterion | What we look for |
|-----------|------------------|
| Correctness | The mview and unique index create cleanly; concurrent refresh runs |
| Constraint mapping | `DESIGN.md` maps each of the 5 constraints to your choice |
| Freshness math | Your interval is justified against the 15-min tolerance |
| Rejected alternative | You explain why the other two strategies lose here |
| Failure mode | You name what breaks (e.g., "if a refresh takes >10 min, refreshes overlap") and how you'd detect it |

## Stretch

- Add a guard so overlapping refreshes can't pile up (advisory lock, or check `pg_stat_activity`).
- Track refresh duration over time in a small `refresh_log` table (a `pg_cron` job can wrap the refresh in a function that records start/end). At what data volume does your chosen interval stop being safe?
- Compare `CONCURRENTLY` vs. plain refresh wall-clock time on your test data and report the ratio.
