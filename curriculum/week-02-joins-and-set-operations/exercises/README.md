# Week 2 — Exercises

Three guided reps. Do them in order — each builds on the last.

| # | Exercise | Skill | Time |
|--:|----------|-------|-----:|
| 1 | [Two-table joins](./exercise-01-two-table-joins.md) | `INNER JOIN`, `ON`, `USING`, aliases | 40m |
| 2 | [Outer joins & NULLs](./exercise-02-outer-joins-and-nulls.md) | `LEFT`/`RIGHT`/`FULL`, `COALESCE`, the `WHERE` trap | 45m |
| 3 | [Self-joins & set ops](./exercise-03-self-joins-and-set-ops.md) | self-join, `UNION`/`INTERSECT`/`EXCEPT` | 45m |

Every exercise ends with a **Done when…** checklist. Write your answers into a
`solutions.sql` file, run each, and paste the row count you got as a comment.

---

## Setup — load the `crunch_shop` seed database (do this ONCE, first)

Every lecture, exercise, challenge, and the mini-project runs against the same
tiny store schema, `crunch_shop`. Load it now. The dataset is deliberately small
(you can hold every row in your head) and deliberately *gappy* — some customers
never ordered, some products were never sold, one employee has no manager, one
customer has no region. Those gaps are what make outer joins and anti-joins
visible.

### The schema

```
regions(region_id, name)
suppliers(supplier_id, name, region_id → regions)
categories(category_id, name)
employees(employee_id, name, manager_id → employees, region_id → regions)
customers(customer_id, name, region_id → regions)
products(product_id, name, category_id → categories, supplier_id → suppliers, price)
orders(order_id, customer_id → customers, employee_id → employees, order_date, status)
order_items(order_id → orders, product_id → products, quantity, unit_price)
```

`order_items` is the classic **junction table**: one order has many items, each
item points at one product. `employees.manager_id` points back at `employees` —
that self-reference is your self-join in Exercise 3.

### PostgreSQL 16

```bash
createdb crunch_shop
psql crunch_shop -f seed.sql        # paste the SQL below into seed.sql first
psql crunch_shop                    # interactive shell
```

### SQLite (zero setup)

```bash
sqlite3 crunch_shop.db < seed.sql   # paste the SQL below into seed.sql first
sqlite3 crunch_shop.db              # interactive shell
# in the shell, for readable output:
.headers on
.mode column
```

### `seed.sql` — copy this whole block

This block runs unmodified on **PostgreSQL 16** and **SQLite 3.39+**.

```sql
DROP TABLE IF EXISTS order_items;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS customers;
DROP TABLE IF EXISTS employees;
DROP TABLE IF EXISTS categories;
DROP TABLE IF EXISTS suppliers;
DROP TABLE IF EXISTS regions;

CREATE TABLE regions (
  region_id   INTEGER PRIMARY KEY,
  name        TEXT NOT NULL
);

CREATE TABLE suppliers (
  supplier_id INTEGER PRIMARY KEY,
  name        TEXT NOT NULL,
  region_id   INTEGER REFERENCES regions(region_id)
);

CREATE TABLE categories (
  category_id INTEGER PRIMARY KEY,
  name        TEXT NOT NULL
);

CREATE TABLE employees (
  employee_id INTEGER PRIMARY KEY,
  name        TEXT NOT NULL,
  manager_id  INTEGER REFERENCES employees(employee_id),
  region_id   INTEGER REFERENCES regions(region_id)
);

CREATE TABLE customers (
  customer_id INTEGER PRIMARY KEY,
  name        TEXT NOT NULL,
  region_id   INTEGER REFERENCES regions(region_id)
);

CREATE TABLE products (
  product_id  INTEGER PRIMARY KEY,
  name        TEXT NOT NULL,
  category_id INTEGER REFERENCES categories(category_id),
  supplier_id INTEGER REFERENCES suppliers(supplier_id),
  price       NUMERIC(10,2) NOT NULL
);

CREATE TABLE orders (
  order_id    INTEGER PRIMARY KEY,
  customer_id INTEGER REFERENCES customers(customer_id),
  employee_id INTEGER REFERENCES employees(employee_id),
  order_date  DATE NOT NULL,
  status      TEXT NOT NULL
);

CREATE TABLE order_items (
  order_id    INTEGER REFERENCES orders(order_id),
  product_id  INTEGER REFERENCES products(product_id),
  quantity    INTEGER NOT NULL,
  unit_price  NUMERIC(10,2) NOT NULL,
  PRIMARY KEY (order_id, product_id)
);

-- 4 regions; region 4 (West) has NO customers and NO employees on purpose.
INSERT INTO regions (region_id, name) VALUES
  (1, 'North'), (2, 'South'), (3, 'East'), (4, 'West');

INSERT INTO suppliers (supplier_id, name, region_id) VALUES
  (1, 'NorthParts', 1),
  (2, 'SouthParts', 2),
  (3, 'EastParts',  3),
  (4, 'WestParts',  4);   -- supplies nothing (no products point here)

INSERT INTO categories (category_id, name) VALUES
  (1, 'Widgets'), (2, 'Gadgets'), (3, 'Gizmos');

-- Ada is CEO: manager_id IS NULL. That NULL drives the self-join outer case.
INSERT INTO employees (employee_id, name, manager_id, region_id) VALUES
  (1, 'Ada',  NULL, 1),
  (2, 'Ben',  1,    1),
  (3, 'Cy',   1,    2),
  (4, 'Dot',  2,    3),
  (5, 'Eve',  3,    2),
  (6, 'Finn', 2,    NULL);   -- no region assigned

-- Customer 6 (Foxtrot) has region NULL; customer 7 (Gizmo Inc) never orders.
INSERT INTO customers (customer_id, name, region_id) VALUES
  (1, 'Acme',       1),
  (2, 'Bolt Co',    1),
  (3, 'Cog Ltd',    2),
  (4, 'Dyeworks',   3),
  (5, 'Echo LLC',   2),
  (6, 'Foxtrot',    NULL),
  (7, 'Gizmo Inc',  1);

-- Product 5 (Widget-X) is never ordered; product 6 (Gizmo-Pro) has no category.
INSERT INTO products (product_id, name, category_id, supplier_id, price) VALUES
  (1, 'Bolt',      1, 1,  2.50),
  (2, 'Nut',       1, 1,  1.25),
  (3, 'Cog',       2, 2,  9.00),
  (4, 'Sprocket',  2, 2, 12.00),
  (5, 'Widget-X',  3, 3, 40.00),
  (6, 'Gizmo-Pro', NULL, NULL, 99.00);

INSERT INTO orders (order_id, customer_id, employee_id, order_date, status) VALUES
  (101, 1, 2, DATE '2026-01-05', 'shipped'),
  (102, 1, 2, DATE '2026-02-10', 'shipped'),
  (103, 2, 3, DATE '2026-01-20', 'pending'),
  (104, 3, 5, DATE '2026-03-01', 'shipped'),
  (105, 4, 4, DATE '2026-03-15', 'cancelled'),
  (106, 5, 5, DATE '2026-04-02', 'shipped'),
  (107, 6, 2, DATE '2026-04-10', 'pending');

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
  (101, 1, 10,  2.50),
  (101, 2, 20,  1.25),
  (102, 3,  2,  9.00),
  (103, 1,  5,  2.50),
  (104, 4,  1, 12.00),
  (104, 3,  3,  9.00),
  (105, 2, 100, 1.20),
  (106, 1,  8,  2.50),
  (106, 3,  1,  9.00),
  (107, 4,  2, 12.00);
```

> **SQLite gotcha:** SQLite accepts `DATE '2026-01-05'` but stores it as the text
> `'2026-01-05'`. Date comparisons still work because ISO-8601 text sorts
> chronologically. PostgreSQL stores a real `date` type. For this week the
> difference does not matter.

### Sanity check

After loading, confirm the gaps are present — you'll use every one of these:

```sql
SELECT count(*) FROM regions;      -- 4
SELECT count(*) FROM customers;    -- 7
SELECT count(*) FROM products;     -- 6
SELECT count(*) FROM orders;       -- 7
SELECT count(*) FROM order_items;  -- 10
```

Known gaps to remember:

| Gap | Table / row | Shows up in |
|-----|-------------|-------------|
| Customer with no orders | `customers` 7 (Gizmo Inc) | anti-join, `LEFT JOIN … IS NULL` |
| Product never ordered | `products` 5 (Widget-X) | anti-join |
| Product with no category | `products` 6 (Gizmo-Pro) | `LEFT JOIN` to `categories` |
| Customer with no region | `customers` 6 (Foxtrot) | `LEFT JOIN` to `regions` |
| Employee with no manager | `employees` 1 (Ada) | self-join outer |
| Region with no people | `regions` 4 (West) | `RIGHT`/`FULL` join, `EXCEPT` |
| Supplier with no products | `suppliers` 4 (WestParts) | anti-join |

If the counts and gaps match, you're ready. Start with
[Exercise 1](./exercise-01-two-table-joins.md).
