# Week 4 — Quiz

Thirteen questions. Lectures closed. Aim for 11/13 before starting Week 5. Mix of multiple-choice and short-answer; the answer key is at the bottom.

---

**Q1.** A *candidate key* is:

- A) Any set of columns that identifies a row, extras allowed.
- B) A minimal set of columns that uniquely identifies a row.
- C) The one key chosen as the row's official identity.
- D) A column that references another table.

---

**Q2.** You choose a surrogate `BIGINT` PK for `customers` even though `email` is unique. What must you *still* do?

- A) Nothing — the surrogate replaces the need for email uniqueness.
- B) Put a `UNIQUE` constraint on `email`.
- C) Make `email` the foreign key.
- D) Drop the `email` column.

---

**Q3.** In a 1:N relationship between `customers` and `orders`, the foreign key goes:

- A) On `customers` (the "one" side).
- B) On `orders` (the "many" side).
- C) In a separate junction table.
- D) On whichever table is created first.

---

**Q4.** Why can't a plain relational table represent a many-to-many relationship directly, and what resolves it? *(short answer)*

---

**Q5.** Which anomaly is this? *"You delete the last enrollment for a course, and in doing so lose the fact that the course's instructor exists at all."*

- A) Update anomaly
- B) Insertion anomaly
- C) Deletion anomaly
- D) Referential anomaly

---

**Q6.** A table's primary key is a single column `id`. Which normal form does it automatically satisfy (given it's already 1NF)?

- A) BCNF
- B) 3NF
- C) 2NF
- D) 4NF

---

**Q7.** The functional dependency `instructor → department` in a table keyed on `course_id` (where `course_id → instructor`) is an example of a ______ dependency, and it violates ______.

- A) partial; 2NF
- B) transitive; 3NF
- C) trivial; BCNF
- D) multivalued; 4NF

---

**Q8.** State the "the key, the whole key, and nothing but the key" mnemonic and map each clause to the normal form it represents. *(short answer)*

---

**Q9.** BCNF differs from 3NF only when a table has:

- A) A single-column primary key.
- B) No foreign keys.
- C) Overlapping candidate keys, with a non-key column determining part of a key.
- D) More than five columns.

---

**Q10.** You have `order_items(order_id FK→orders, product_id FK→products, ...)`. Deleting an *order* should remove its items; deleting a *product* that's been sold should be blocked. The correct `ON DELETE` actions are:

- A) both `CASCADE`
- B) `CASCADE` on `order_id`, `RESTRICT`/`NO ACTION` on `product_id`
- C) `RESTRICT` on `order_id`, `CASCADE` on `product_id`
- D) both `SET NULL`

---

**Q11.** Why should you use `NUMERIC(10,2)` rather than `DOUBLE PRECISION` for a money column? *(short answer)*

---

**Q12.** Storing `order_items.price_at_sale` (a copy of the product's price at purchase time) is:

- A) A 2NF violation to be normalized away.
- B) A transitive dependency.
- C) A snapshot of a point-in-time fact — correct modeling, not denormalization.
- D) An insertion anomaly.

---

**Q13.** In SQLite, what one statement must you run every connection to get foreign-key enforcement, and why isn't it needed in PostgreSQL? *(short answer)*

---

## Answer key

<details>
<summary>Reveal after attempting</summary>

1. **B** — a *minimal* uniquely-identifying set. (A is a superkey; C is the primary key; D is a foreign key.)
2. **B** — keep the `UNIQUE` on `email`. A surrogate PK gives stability, but the *real* uniqueness rule (no two customers with one email) must still be enforced, or you get duplicate customers.
3. **B** — the FK lives on the "many" side (`orders.customer_id`). The "many" is expressed by many rows sharing one `customer_id`.
4. A relational row has fixed columns, so there's nowhere to store "many" foreign keys on either side. You **resolve it with a junction table** holding two FKs (one to each entity) and typically a composite PK — turning one M:N into two 1:N relationships.
5. **C** — deletion anomaly (deleting one kind of row destroys an unrelated fact that lived only in that row).
6. **C** — 2NF. A partial dependency requires a *composite* key to be partial *of*; a single-column PK can't have one, so 1NF ⇒ automatically 2NF. (It is *not* automatically 3NF — a transitive dependency is still possible.)
7. **B** — transitive dependency; violates 3NF (a non-key column depends on another non-key column).
8. *"The key"* = a primary key exists / values are atomic (1NF-adjacent). *"The whole key"* = 2NF (no non-key column depends on only part of a composite key). *"Nothing but the key"* = 3NF (no non-key column depends on another non-key column). BCNF tightens it to "every determinant is a key."
9. **C** — overlapping composite candidate keys where a non-key column determines part of a key. Otherwise 3NF and BCNF coincide.
10. **B** — `CASCADE` on the order link (items are *part of* the order), `RESTRICT`/`NO ACTION` on the product link (don't let a sold product be deleted and destroy history).
11. `DOUBLE PRECISION` is binary floating point and cannot represent decimal values like `0.10` exactly, so sums drift by fractions of a cent. `NUMERIC` is exact decimal arithmetic — required for money.
12. **C** — it's a snapshot of a *different* fact ("price at the moment of sale"), independent of the current price. It records history correctly; it is not a duplicate of a single fact and cannot drift, so it's not denormalization.
13. `PRAGMA foreign_keys = ON;` — SQLite leaves FK enforcement *off* by default for backward compatibility, and the setting is per-connection. PostgreSQL always enforces foreign keys, so there's nothing to turn on.

</details>

**Scoring:** 11+ → start Week 5. 8–10 → re-read the lecture behind each miss. <8 → re-read Lectures 2 and 3 from the top before continuing.
