# Challenge 1 — Pick the Right Isolation Level

**Type:** open-ended · **Time:** ~1 hour · **Deliverable:** `writeup.md`

Isolation level is a decision, not a default. For each of the four scenarios below, choose `READ COMMITTED`, `REPEATABLE READ`, or `SERIALIZABLE` (Postgres's three real levels), and **defend it**: name the anomaly you're defending against, name the concurrency cost you're accepting, and say what *else* (a lock, a constraint, a retry loop) you'd add to make the choice safe. There is no single correct answer — but there are indefensible ones.

## The scenarios

### Scenario A — Single-row balance debit

A payments service runs, thousands of times per second:

```sql
UPDATE accounts SET balance = balance - :amount
WHERE id = :id AND balance >= :amount;
```

One statement, in-place arithmetic, guarded by a `WHERE`. Which level? What makes this the *easy* case — why don't the READ-COMMITTED anomalies bite here?

### Scenario B — End-of-day financial report

A batch job runs 30 separate `SELECT`s — totals, subtotals, reconciliations — that must all reflect **the same instant**. If new transactions commit while the report runs, the subtotals won't add up to the totals. The job is read-only and takes ~4 minutes. Which level? What does it cost to hold one snapshot for 4 minutes, and is that acceptable for a read-only job?

### Scenario C — Seat booking with a capacity limit

An event has 100 seats. Each booking does: count current bookings, and if `< 100`, insert a new booking row. Under load, you must never sell seat #101. This is a **multi-row invariant** (the count spans many rows). Which level? If you pick something below SERIALIZABLE, what *other* mechanism enforces the limit — and could you make the invariant single-row instead (hint: a `seats_taken` counter with a `CHECK`)? Discuss at least two designs and pick one.

### Scenario D — Analytics dashboard, "roughly live"

A public dashboard shows approximate counts ("~14,000 signups today") refreshed every 10 seconds. Slightly stale or slightly inconsistent numbers are completely fine; latency and load are the priorities. Which level? Why is reaching for a strong level here a *mistake*?

## Constraints

- You must pick exactly one level per scenario and justify it in **3–5 sentences** each.
- For any scenario where you pick below SERIALIZABLE but the scenario has an invariant, you **must** name the additional safeguard (lock / constraint / atomic write / retry) that keeps it correct. "REPEATABLE READ and hope" is not an answer.
- For any scenario where you pick SERIALIZABLE, you **must** state that a retry loop is required and why.

## Hints

<details>
<summary>How to reason about each one</summary>

Ask three questions per scenario, in order:

1. **Is the correctness condition single-row or multi-row?** Single-row atomic writes (`SET x = x - n WHERE …`) are safe at READ COMMITTED because the `UPDATE` re-reads the current row under a row lock. Multi-row invariants (a count, a sum across rows, "at least one of…") are where write skew lives and where you need SERIALIZABLE *or* an explicit safeguard.
2. **Do I need many statements to see one consistent world?** If yes → at least REPEATABLE READ (one snapshot).
3. **What's the cost?** Higher levels mean longer-held snapshots (more bloat, longer vacuum lag) and, at SERIALIZABLE, `40001` aborts that force retries. Don't buy protection you don't need.

</details>

<details>
<summary>The trap in Scenario C</summary>

Counting rows then inserting is the textbook write-skew shape — two bookings each count 99 and each insert, giving 101. SERIALIZABLE catches it (with retries). But often the *better* fix is to **remove the multi-row-ness**: keep a single `events(seats_taken, capacity)` row with `CHECK (seats_taken <= capacity)` and do `UPDATE events SET seats_taken = seats_taken + 1 WHERE id=:e AND seats_taken < capacity` — now it's a safe single-row atomic write at READ COMMITTED. Recognizing when to reshape the data to avoid needing strong isolation is the senior move.

</details>

## How success is judged

| Criterion | Weak | Strong |
|-----------|------|--------|
| Level choice | Picks a level with no reason, or "SERIALIZABLE everywhere to be safe" | Picks the *lowest* level that's correct for the scenario |
| Anomaly named | Doesn't say what could go wrong | Names the specific anomaly each choice defends against |
| Cost named | Ignores the downside | States the concurrency/latency/retry cost accepted |
| Safeguards | "Set the level and hope" | Adds the right lock/constraint/atomic-write/retry where needed |
| Reshaping (C) | Only reaches for isolation | Recognizes the counter+CHECK redesign that makes strong isolation unnecessary |

Full marks = every scenario has a defensible level, the anomaly it fights, the cost it pays, and (where relevant) the extra safeguard — and Scenario C shows you can redesign the data to dodge the problem.

## Deliverable

`writeup.md` with your four decisions and justifications. Commit to `c33-week-05/challenge-01/`.
