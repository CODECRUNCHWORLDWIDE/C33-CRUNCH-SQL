# Mini-Project — Design & Create a Normalized Schema From a Spec

> Take a written product spec, model it, normalize it to 3NF, and build it as a fully-constrained, runnable PostgreSQL 16 schema — the whole week's skill in one deliverable.

**Estimated time:** 5–6 hours, spread across the back half of the week.

This is the capstone of Week 4. You will produce a real, runnable database from a paragraph of English — the exact task every backend engineer does at the start of every project. It ties together all three lectures: keys and relationships (L1), the normal forms (L2), and constrained DDL (L3).

---

## The spec

You're building the database for **"Crunch Eats"**, a small food-delivery marketplace.

> **Restaurants** join the platform; each has a name, an address, a cuisine type, and an owner (one owner may own several restaurants). Each restaurant offers a **menu** of items; a menu item has a name, a description, a price, and belongs to a category (appetizer, main, dessert, drink). **Customers** have a name, an email, a phone, and one or more saved **delivery addresses**. A customer places an **order** from exactly one restaurant; an order contains one or more menu items, each with a quantity, and captures the price of each item *at the time of ordering*. An order has a status (placed, preparing, out-for-delivery, delivered, cancelled), a delivery address, an assigned **courier** (a courier can deliver many orders), and a timestamp. After delivery a customer may leave one **review** per order — a 1–5 star rating and optional text. The platform charges each restaurant a commission percentage that can differ per restaurant.

Read it three times. Underline the nouns (entities) and the verbs (relationships). Note every "one or more" (a 1:N or M:N) and every "exactly one" (a 1:1 or the "one" side of a 1:N).

---

## Deliverables

A directory `c33-week-04/mini-project/` containing:

1. **`er-model.md`** — the entity-relationship diagram (Mermaid or an exported image) with every entity, its keys, and cardinality-annotated relationships.
2. **`schema.sql`** — the complete, runnable DDL: `CREATE TABLE`s in dependency order, full constraints, idempotent (`DROP TABLE IF EXISTS ... CASCADE` header in reverse order).
3. **`seed.sql`** — enough sample data to make the schema real: ≥3 restaurants, ≥5 menu items each, ≥4 customers, ≥6 orders across several statuses, a couple of couriers, and a few reviews.
4. **`queries.sql`** — five `SELECT`s that *prove the model works* (listed under Milestone 4).
5. **`design-notes.md`** — a short memo defending your key choices, `ON DELETE` actions, and any deliberate denormalization.

---

## Milestones

### Milestone 1 — Model (1.5h)

Identify the entities. Expect around **9–11 tables** after resolving many-to-manys. The tricky ones:

- `order_items` is the junction resolving the order↔menu-item M:N, and it's where `quantity` and `price_at_order` live.
- `delivery_addresses` is a 1:N off customer (a customer has *several*) — not columns on `customers`.
- `reviews` is 1:1 with `orders` ("one review per order") — enforce it.
- `owners`/`couriers` are their own entities (an owner has many restaurants; a courier has many orders).

Draw the ERD in `er-model.md` before writing any SQL.

### Milestone 2 — Normalize (1h)

Walk your model against the ladder and write a paragraph in `design-notes.md` confirming it's 3NF: no repeating groups (1NF), no partial dependencies (2NF), no transitive dependencies (3NF). Explicitly call out `price_at_order` as a *snapshot*, not a normalization violation — and explain why.

### Milestone 3 — Build (1.5h)

Write `schema.sql`. Every table gets a surrogate PK (`BIGINT GENERATED ALWAYS AS IDENTITY`). Every FK gets a deliberate `ON DELETE`. Enforce, at minimum:

- `email` UNIQUE on customers; restaurant-owner and courier links as FKs.
- `price >= 0`, `quantity > 0`, `rating BETWEEN 1 AND 5`, `commission_pct BETWEEN 0 AND 100` — all CHECKs.
- `status` and `category` bounded by CHECK.
- One review per order (a `UNIQUE` on `reviews.order_id`, or make it the PK).
- Run it into an empty database until it creates with zero errors.

### Milestone 4 — Prove it (1h)

Write `seed.sql`, load it, then write these five queries in `queries.sql` and confirm each returns sensible rows:

1. Every menu item for a given restaurant, with category and price.
2. A given customer's full order history: order date, restaurant, status, and order total (sum of `quantity * price_at_order`).
3. The top 3 restaurants by number of delivered orders.
4. Average rating per restaurant (join reviews → orders → restaurants).
5. Every order currently `out-for-delivery`, with the courier's name and the delivery address.

If any query is awkward or impossible, your model has a flaw — go fix the model, not the query.

### Milestone 5 — Defend (0.5h)

Finish `design-notes.md`: justify surrogate-vs-natural key choices, each `ON DELETE` action (which relationships are "part-of" → CASCADE, which are "refers-to" → RESTRICT), and whether you denormalized anything (e.g., a stored `order_total`) and why.

---

## Acceptance criteria

- [ ] `schema.sql` creates cleanly in an empty PostgreSQL 16 database, is idempotent, and has ≥9 tables.
- [ ] Every table has a primary key; every foreign key has an explicit `ON DELETE` action.
- [ ] Every M:N (order↔item) is a junction table; every "one or more"-off-an-entity (addresses) is a 1:N table, not repeated columns.
- [ ] All bounded values (`status`, `category`, `rating`, `commission_pct`, prices, quantities) are enforced by CHECK/UNIQUE.
- [ ] `reviews` enforces exactly one review per order.
- [ ] All five proof queries run and return correct results against `seed.sql`.
- [ ] `design-notes.md` argues the schema is 3NF and defends the key/`ON DELETE`/denormalization decisions.

---

## Rubric

| Criterion | Weight | "Great" looks like |
|-----------|-------:|--------------------|
| Model correctness | 25% | Every fact in the spec is storable; cardinalities are right. |
| Normalization | 20% | Clean 3NF; snapshot vs. duplicate understood and stated. |
| Constraints & integrity | 25% | Illegal states are *impossible*, not merely discouraged. |
| Proof queries | 15% | All five run; totals and joins are correct. |
| Defensibility | 15% | Notes justify keys, `ON DELETE`, and denormalization trade-offs. |

---

## Why this matters

This is the deliverable you'll reuse. The schema you design here is exactly the kind [C16 Crunch Pro Web Backend](../../../C16-CRUNCH-PRO-WEB-BACKEND/) builds an API on top of, and the normalized structure is what makes Week 6's indexing and Week 7's query tuning *tractable* — you can't tune a query against a god-table. A clean model is the foundation everything later in C33 stands on.

When it runs clean and all five queries pass, commit it and move to [Week 5 — Transactions, ACID & MVCC](../../week-05-transactions-acid-and-mvcc/).
