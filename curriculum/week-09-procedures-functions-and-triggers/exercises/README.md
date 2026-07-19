# Week 9 — Exercises

Three guided reps, one per lecture. Do each the same day you read its lecture — the point is to move the concept from "I read it" to "my fingers know it." All exercises run on **PostgreSQL 16** against the course seed schema.

## Setup — the practice schema

Every exercise assumes this small schema. Create it once in a scratch database and reuse it:

```sql
CREATE TABLE customers (
    customer_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    full_name   text        NOT NULL,
    email       text        NOT NULL UNIQUE,
    country     text        NOT NULL,
    status      text        NOT NULL DEFAULT 'active',
    created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE orders (
    order_id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id bigint      NOT NULL REFERENCES customers(customer_id),
    status      text        NOT NULL DEFAULT 'pending',
    placed_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
    order_item_id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id         bigint NOT NULL REFERENCES orders(order_id),
    product_name     text   NOT NULL,
    quantity         int    NOT NULL CHECK (quantity > 0),
    unit_price_cents bigint NOT NULL CHECK (unit_price_cents >= 0)
);

INSERT INTO customers(full_name, email, country) VALUES
    ('Ada Lovelace','ada@example.com','GB'),
    ('Grace Hopper','grace@example.com','US'),
    ('Dennis Ritchie','dennis@example.com','US'),
    ('Barbara Liskov','barbara@example.com','US');

INSERT INTO orders(customer_id, status) VALUES
    (1,'paid'), (1,'paid'), (2,'paid'), (2,'pending'), (3,'paid');

INSERT INTO order_items(order_id, product_name, quantity, unit_price_cents) VALUES
    (1,'Keyboard',1,7999), (1,'Mouse',2,2499),
    (2,'Monitor',1,19999),
    (3,'Cable',3,599),  (3,'Dock',1,14999),
    (5,'Webcam',1,4999);
```

## The exercises

| # | File | Drills | Time |
|---|------|--------|------|
| 1 | [exercise-01-view-and-sql-function.md](./exercise-01-view-and-sql-function.md) | A view + a `LANGUAGE sql` function (Lecture 1 & 2) | ~40m |
| 2 | [exercise-02-plpgsql-function-control-flow.md](./exercise-02-plpgsql-function-control-flow.md) | A PL/pgSQL function with `IF`/loop + `EXCEPTION` (Lecture 2) | ~50m |
| 3 | [exercise-03-audit-trigger.md](./exercise-03-audit-trigger.md) | An `AFTER ROW` audit trigger (Lecture 3) | ~50m |

## How to submit

For each exercise, save your SQL in a file named after the exercise (e.g. `exercise-01.sql`) plus a short `notes.md` answering its questions. Commit them to your portfolio under `c33-week-09/`.

Everything is runnable — no theory-only answers. If a statement errors, read the error; PostgreSQL's messages are unusually good, and decoding them is part of the skill.
