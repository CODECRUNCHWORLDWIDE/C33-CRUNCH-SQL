# Week 3 — Exercises

Three guided reps, ~40–60 min each. **Type the queries, don't paste.** Each exercise says exactly when you're done.

1. **[Exercise 1 — Group and aggregate](exercise-01-group-and-aggregate.md)** — `COUNT`/`SUM`/`AVG`, `GROUP BY`, `HAVING`, `FILTER`.
2. **[Exercise 2 — Subqueries](exercise-02-subqueries.md)** — scalar, `IN`, `EXISTS`, and correlated subqueries.
3. **[Exercise 3 — CTEs](exercise-03-ctes.md)** — chained `WITH` blocks plus one recursive CTE.

Run them in order — each assumes the seed database from the setup below.

---

## Setup — load the `crunch_shop` seed database (do this once)

Every lecture, exercise, challenge, and the mini-project this week run against the same tiny store database, `crunch_shop`. Load it now.

### PostgreSQL 16

```bash
createdb crunch_shop
psql crunch_shop -f seed.sql     # or paste the SQL below into psql
```

### SQLite

```bash
sqlite3 crunch_shop.db < seed.sql
```

The seed SQL below is standard and runs on **both** engines. Save it as `seed.sql`, or paste it directly into your client.

```sql
-- ---------- schema ----------
DROP TABLE IF EXISTS order_items;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS categories;
DROP TABLE IF EXISTS customers;
DROP TABLE IF EXISTS employees;

CREATE TABLE customers (
    customer_id  INTEGER PRIMARY KEY,
    full_name    TEXT NOT NULL,
    country      TEXT NOT NULL,
    signup_date  DATE NOT NULL
);

CREATE TABLE categories (
    category_id  INTEGER PRIMARY KEY,
    name         TEXT NOT NULL,
    parent_id    INTEGER REFERENCES categories(category_id)
);

CREATE TABLE products (
    product_id   INTEGER PRIMARY KEY,
    name         TEXT NOT NULL,
    category_id  INTEGER REFERENCES categories(category_id),
    price        NUMERIC(10,2) NOT NULL
);

CREATE TABLE orders (
    order_id     INTEGER PRIMARY KEY,
    customer_id  INTEGER NOT NULL REFERENCES customers(customer_id),
    order_date   DATE NOT NULL,
    status       TEXT NOT NULL          -- paid | shipped | refunded | cancelled
);

CREATE TABLE order_items (
    order_id     INTEGER NOT NULL REFERENCES orders(order_id),
    product_id   INTEGER NOT NULL REFERENCES products(product_id),
    quantity     INTEGER NOT NULL,
    unit_price   NUMERIC(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);

CREATE TABLE employees (
    employee_id  INTEGER PRIMARY KEY,
    full_name    TEXT NOT NULL,
    manager_id   INTEGER REFERENCES employees(employee_id),
    hire_date    DATE NOT NULL
);

-- ---------- customers ----------
INSERT INTO customers (customer_id, full_name, country, signup_date) VALUES
 (1, 'Ana Ruiz',      'US', '2024-01-05'),
 (2, 'Ben Okafor',    'US', '2024-02-11'),
 (3, 'Chen Wei',      'CA', '2024-02-19'),
 (4, 'Dana Ito',      'CA', '2024-03-02'),
 (5, 'Elif Demir',    'DE', '2024-03-15'),
 (6, 'Farah Nasser',  'DE', '2024-04-01'),
 (7, 'Gus Lindqvist', 'SE', '2024-04-22'),
 (8, 'Hana Kim',      'US', '2024-05-09');   -- has never ordered (on purpose)

-- ---------- categories (self-referencing tree) ----------
INSERT INTO categories (category_id, name, parent_id) VALUES
 (1, 'All',          NULL),
 (2, 'Electronics',  1),
 (3, 'Computers',    2),
 (4, 'Laptops',      3),
 (5, 'Accessories',  2),
 (6, 'Home',         1),
 (7, 'Kitchen',      6);

-- ---------- products ----------
INSERT INTO products (product_id, name, category_id, price) VALUES
 (1, 'Featherweight Laptop', 4, 1299.00),
 (2, 'Workhorse Laptop',     4,  899.00),
 (3, 'Mechanical Keyboard',  5,  120.00),
 (4, 'USB-C Hub',            5,   45.00),
 (5, 'Noise-Cancel Headset', 5,  199.00),
 (6, 'Desk Lamp',            7,   38.00),
 (7, 'Chef Knife',           7,   85.00),
 (8, 'Ceramic Mug',          7,   14.00),
 (9, 'Unsold Monitor',       3,  240.00);   -- never ordered (on purpose)

-- ---------- orders ----------
INSERT INTO orders (order_id, customer_id, order_date, status) VALUES
 (1, 1, '2024-06-01', 'paid'),
 (2, 1, '2024-06-15', 'paid'),
 (3, 2, '2024-06-03', 'shipped'),
 (4, 3, '2024-06-07', 'paid'),
 (5, 3, '2024-06-20', 'refunded'),
 (6, 4, '2024-06-09', 'paid'),
 (7, 5, '2024-06-11', 'cancelled'),
 (8, 5, '2024-06-25', 'paid'),
 (9, 6, '2024-06-18', 'paid'),
 (10,7, '2024-06-28', 'shipped');

-- ---------- order_items ----------
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
 (1, 1, 1, 1299.00),
 (1, 3, 1,  120.00),
 (2, 4, 2,   45.00),
 (3, 2, 1,  899.00),
 (4, 5, 1,  199.00),
 (4, 3, 1,  120.00),
 (5, 1, 1, 1299.00),
 (6, 7, 2,   85.00),
 (6, 8, 4,   14.00),
 (7, 2, 1,  899.00),
 (8, 6, 1,   38.00),
 (8, 8, 3,   14.00),
 (9, 5, 1,  199.00),
 (9, 4, 1,   45.00),
 (10,7, 1,   85.00);

-- ---------- employees (self-referencing org chart) ----------
INSERT INTO employees (employee_id, full_name, manager_id, hire_date) VALUES
 (1, 'Root Boss',   NULL, '2020-01-01'),
 (2, 'Vera Lopez',   1,   '2021-03-01'),
 (3, 'Wole Adeyemi', 1,   '2021-06-15'),
 (4, 'Dana Ito',     2,   '2022-02-01'),
 (5, 'Sam Park',     2,   '2022-09-10'),
 (6, 'Tara Singh',   4,   '2023-04-20');
```

### Sanity check

```sql
SELECT (SELECT COUNT(*) FROM customers)   AS customers,   -- 8
       (SELECT COUNT(*) FROM products)    AS products,    -- 9
       (SELECT COUNT(*) FROM orders)      AS orders,       -- 10
       (SELECT COUNT(*) FROM order_items) AS items;        -- 15
```

If those four numbers match the comments, you're ready.

## Suggested workflow

- Keep `psql` (or `sqlite3`) open in one pane, this file in your editor in another.
- Write each query, run it, and **compare against the expected result** the exercise gives.
- If a result surprises you, don't move on — figure out *why* before continuing. That "why" is the learning.
- Save your final answers to a `solutions.sql` you commit to your portfolio under `c33-week-03/`.
