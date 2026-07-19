# Exercise 3 — Constraints and Safe Queries

**Goal:** Two halves. First, add integrity constraints — including an `EXCLUDE` constraint that stops double-booking — and watch the engine reject bad data. Second, take a deliberately injectable query, *exploit* it to prove it's broken, then rewrite it as a parameterized query and confirm the exploit dies.

**Estimated time:** ~2 hours.

## Part A — Constraints

### Setup

```sql
CREATE TABLE product (
  id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name  text NOT NULL,
  price numeric(10,2) NOT NULL
);

CREATE EXTENSION IF NOT EXISTS btree_gist;   -- needed to mix = and && in one index
CREATE TABLE booking (
  id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  room_id int NOT NULL,
  guest   text NOT NULL,
  during  tstzrange NOT NULL
);
```

### Steps

1. **Add a `CHECK`.** Forbid non-positive prices:
   ```sql
   ALTER TABLE product ADD CONSTRAINT price_positive CHECK (price > 0);
   ```
   Prove it: `INSERT INTO product (name, price) VALUES ('Widget', -5);` must fail. A `price` of `0.01` must succeed.

2. **Add a multi-column `CHECK`** to `booking` requiring a non-empty, forward range:
   ```sql
   ALTER TABLE booking ADD CONSTRAINT range_forward CHECK (NOT isempty(during) AND lower(during) < upper(during));
   ```

3. **Add the `EXCLUDE` constraint** — no two bookings for the same room may overlap in time:
   ```sql
   ALTER TABLE booking ADD CONSTRAINT no_double_book
     EXCLUDE USING gist (room_id WITH =, during WITH &&);
   ```

4. **Prove the `EXCLUDE` works:**
   ```sql
   INSERT INTO booking (room_id, guest, during)
     VALUES (7, 'Ada',   '[2026-08-01 09:00, 2026-08-01 11:00)');   -- ok
   INSERT INTO booking (room_id, guest, during)
     VALUES (7, 'Grace', '[2026-08-01 10:00, 2026-08-01 12:00)');   -- FAILS: overlaps
   INSERT INTO booking (room_id, guest, during)
     VALUES (7, 'Grace', '[2026-08-01 11:00, 2026-08-01 12:00)');   -- ok: touches but no overlap
   INSERT INTO booking (room_id, guest, during)
     VALUES (9, 'Grace', '[2026-08-01 10:00, 2026-08-01 12:00)');   -- ok: different room
   ```
   The second insert must raise `conflicting key value violates exclusion constraint "no_double_book"`. Confirm the third (adjacent, half-open `[)` ranges don't overlap) and fourth (different room) both succeed.

## Part B — Fix an injectable query

Below is a small Python lookup (psycopg 3). It's the kind of code that ships in a hurry.

```python
def find_products(conn, search, sort):
    sql = "SELECT id, name, price FROM product " \
          "WHERE name LIKE '%" + search + "%' " \
          "ORDER BY " + sort
    with conn.cursor() as cur:
        cur.execute(sql)                 # <-- string-built SQL
        return cur.fetchall()
```

### Steps

5. **Exploit it (on paper or in a scratch script).** Show what `sql` becomes when `search = "'; DROP TABLE product; --"`. Write the resulting SQL string and explain in one sentence why the database can't tell the injected part from legitimate query text.

6. **Rewrite the value with a parameter.** The `search` value must be bound, not concatenated. Use a placeholder and pass the `%`-wrapping as data:
   ```python
   cur.execute(
       "SELECT id, name, price FROM product WHERE name LIKE %s",
       ["%" + search + "%"],           # the value is data; the % lives in the argument
   )
   ```

7. **Handle `sort` — the identifier placeholders can't reach.** `ORDER BY %s` does **not** work (placeholders bind values, not column names). Validate against an allow-list instead:
   ```python
   SORT_COLUMNS = {"name", "price", "id"}
   if sort not in SORT_COLUMNS:
       raise ValueError("invalid sort column")
   sql = f"SELECT id, name, price FROM product WHERE name LIKE %s ORDER BY {sort}"
   cur.execute(sql, ["%" + search + "%"])
   ```

8. **Confirm the exploit is dead.** With the parameterized version, calling `find_products(conn, "'; DROP TABLE product; --", "name")` must return **zero rows** (no product is literally named that) and leave the table intact — not execute a second statement.

9. **Do the psql/PL/pgSQL version.** Show the same safe lookup as a prepared statement, proving the fix isn't language-specific:
   ```sql
   PREPARE find_prod (text) AS
     SELECT id, name, price FROM product WHERE name LIKE $1;
   EXECUTE find_prod('%; DROP TABLE product; --%');   -- pure data, table survives
   ```

## Done when…

- [ ] `product` rejects non-positive prices; `booking` rejects overlapping same-room ranges but allows adjacent and different-room ranges.
- [ ] You can state, in the resulting SQL string, exactly why the original `find_products` is injectable.
- [ ] The rewrite binds `search` as a parameter and validates `sort` against an allow-list — never concatenating user input into the query text.
- [ ] The DROP-TABLE payload returns zero rows and does not drop the table in either the Python or the `PREPARE` version.

## Stretch

- Add a validation **trigger** enforcing "a product's price may only ever increase" (needs the OLD row — a `CHECK` can't see it). Test an attempted price cut is rejected.
- Explain why the `EXCLUDE` constraint is race-safe under two concurrent overlapping inserts, whereas a `SELECT`-then-`INSERT` in application code is not (tie it back to Week 5).
