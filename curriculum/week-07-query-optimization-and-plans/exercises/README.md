# Week 7 — Exercises

Three guided reps. Each drills one core skill from the lectures against a real, multi-million-row table. Do them in order; each assumes the last.

Before you start, build the practice dataset once. Everything this week runs against it.

```sql
-- setup.sql — run once in psql against a scratch database
DROP TABLE IF EXISTS orders, customers CASCADE;

CREATE TABLE customers (
  id       int PRIMARY KEY,
  country  text NOT NULL,
  city     text NOT NULL,
  tier     text NOT NULL
);

INSERT INTO customers (id, country, city, tier)
SELECT g,
       (ARRAY['US','US','US','CA','GB','FR','DE'])[1 + (g % 7)],
       (ARRAY['NYC','LA','SF','Toronto','London','Paris','Berlin'])[1 + (g % 7)],
       (ARRAY['free','free','free','pro','enterprise'])[1 + (g % 5)]
FROM generate_series(1, 100000) g;

CREATE TABLE orders (
  id           bigint PRIMARY KEY,
  customer_id  int NOT NULL REFERENCES customers(id),
  status       text NOT NULL,
  total        numeric(10,2) NOT NULL,
  created_at   timestamptz NOT NULL
);

INSERT INTO orders (id, customer_id, status, total, created_at)
SELECT g,
       1 + (g % 100000),
       (ARRAY['shipped','shipped','shipped','pending','cancelled'])[1 + (g % 5)],
       round((random() * 500)::numeric, 2),
       timestamptz '2024-01-01' + (g % 900) * interval '1 day'
FROM generate_series(1, 3000000) g;

CREATE INDEX orders_customer_id_idx ON orders (customer_id);
CREATE INDEX orders_created_at_idx  ON orders (created_at);
CREATE INDEX orders_status_idx      ON orders (status);

ANALYZE;
```

That's 3,000,000 orders across 100,000 customers — big enough that plan choices actually matter and timings are honest.

| # | Exercise | Skill | Time |
|--:|----------|-------|------|
| 1 | [exercise-01-read-real-plans.md](./exercise-01-read-real-plans.md) | Read and interpret five real plans in words | ~35m |
| 2 | [exercise-02-compare-join-algorithms.md](./exercise-02-compare-join-algorithms.md) | Force each join algorithm; compare cost + time | ~40m |
| 3 | [exercise-03-fix-a-mis-estimated-plan.md](./exercise-03-fix-a-mis-estimated-plan.md) | Find a bad estimate and fix it with statistics | ~40m |

For every exercise, keep a `solutions.md` with the command you ran, the plan you got, and your written interpretation. The *writing* is the point — a plan you can't explain in words is a plan you can't tune.

## Submission

Commit `setup.sql` and one `solutions.md` per exercise under `c33-week-07/exercises/` in your portfolio.
