# Challenge 2 — Enforce a Business Rule with a Trigger

**Time:** ~75 minutes. **Difficulty:** Medium. **No single right answer.**

## Scenario

A conference-room booking system has a `bookings` table:

```sql
CREATE TABLE rooms (
    room_id int GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name    text NOT NULL
);

CREATE TABLE bookings (
    booking_id  bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id     int         NOT NULL REFERENCES rooms(room_id),
    booked_by   text        NOT NULL,
    starts_at   timestamptz NOT NULL,
    ends_at     timestamptz NOT NULL,
    CHECK (ends_at > starts_at)
);
```

The business rule: **no two bookings for the same room may overlap in time.** A `CHECK` constraint can't express this — it only sees one row at a time, and this rule is about the *relationship between rows*. That is precisely when a trigger earns its place (Lecture 3, Section 8).

Your job: enforce "no overlapping bookings per room" so it **cannot be bypassed**, no matter who writes.

## Constraints

- The rule must hold for both `INSERT` and `UPDATE`.
- A booking that merely *touches* another (one ends exactly when the next starts) is **allowed** — back-to-back is fine.
- Reject a violating write with a clear error message naming the conflict.
- The check must be correct under concurrency (two people booking the same slot at the same instant must not both succeed).

## Your deliverables

1. A `BEFORE INSERT OR UPDATE` trigger function on `bookings` that raises an exception if the new/updated booking overlaps an existing one for the same room.
2. The trigger binding.
3. Test cases in your SQL file proving: a clean insert works, an overlapping insert is rejected, a back-to-back insert is allowed, and an `UPDATE` that creates an overlap is rejected.
4. A `DESIGN.md` addressing the concurrency question (see hints).

## Hints

<details>
<summary>Detecting overlap</summary>

Two ranges `[a_start, a_end)` and `[b_start, b_end)` overlap iff `a_start < b_end AND b_start < a_end`. In the trigger, search for any *other* booking in the same room satisfying that. Exclude the row itself on `UPDATE` (`booking_id <> NEW.booking_id`).

```sql
IF EXISTS (
    SELECT 1 FROM bookings b
    WHERE b.room_id = NEW.room_id
      AND b.booking_id <> NEW.booking_id
      AND NEW.starts_at < b.ends_at
      AND b.starts_at   < NEW.ends_at
) THEN
    RAISE EXCEPTION 'room % is already booked in that window', NEW.room_id
        USING ERRCODE = 'exclusion_violation';
END IF;
```

Note the strict `<` — that's what makes back-to-back (`ends_at = starts_at`) legal.
</details>

<details>
<summary>The concurrency catch</summary>

A `BEFORE` trigger that only does a `SELECT` has a **race**: two concurrent transactions both check, both see no conflict, both insert. The trigger alone doesn't serialize them. Options to discuss in `DESIGN.md`:

1. A PostgreSQL **exclusion constraint** with a `tstzrange` and `&&` (the *native* way to do this — arguably better than a trigger):
   ```sql
   CREATE EXTENSION IF NOT EXISTS btree_gist;
   ALTER TABLE bookings ADD CONSTRAINT no_overlap
       EXCLUDE USING gist (room_id WITH =, tstzrange(starts_at, ends_at) WITH &&);
   ```
2. A **per-room advisory lock** taken at the top of the trigger so bookings for one room serialize.
3. Serializable isolation, retrying on conflict.

Part of this challenge is realizing that for *this* rule, Postgres has a purpose-built feature (the exclusion constraint) — so a strong answer implements the trigger **and** argues whether the exclusion constraint is the better tool. Both viewpoints can score full marks if defended.
</details>

## How success is judged

| Criterion | What we look for |
|-----------|------------------|
| Correctness | Overlaps rejected, back-to-back allowed, `UPDATE` covered |
| Boundary handling | The `<` vs `<=` decision is deliberate and tested |
| Concurrency | `DESIGN.md` acknowledges the race and proposes a real fix |
| Tool judgement | You compare the trigger to the native exclusion constraint |
| Error quality | The rejection message identifies the conflict |

## Stretch

- Implement the exclusion-constraint version *as well* and benchmark inserting 100k non-overlapping bookings under each. Which is faster? Why?
- Extend the rule: a room may be "double-booked" only if one booking is marked `tentative`. How does that change the trigger vs. the exclusion constraint's viability?
