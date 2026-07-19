# Lecture 3 — Set Operations: UNION, INTERSECT, EXCEPT

> **Duration:** ~2 hours. **Outcome:** You can stack query results with `UNION`, `UNION ALL`, `INTERSECT`, and `EXCEPT`; you know the column and type rules, the `ALL` vs distinct semantics, and when a set operator is clearer than a join.

Joins combine tables **side by side** — they add *columns*, gluing a customer's name onto their order. Set operations combine query results **top to bottom** — they stack *rows*, treating two result sets as mathematical sets and taking their union, intersection, or difference. Different question, different tool. This lecture is the second half of "combining data," and it closes with the judgment call: when do you reach for a set operator instead of a join?

## 1. The shape of a set operation

Every set operation has the same skeleton: two (or more) `SELECT` statements joined by a keyword.

```sql
SELECT ... FROM ...
<SET OPERATOR>
SELECT ... FROM ...;
```

For that to make sense, the two `SELECT`s must be **union-compatible**:

1. **Same number of columns**, in the same left-to-right order.
2. **Compatible types**, column by column (int with int, text with text; the engine will coerce close types like `int`/`numeric`).
3. The **column names of the result come from the first `SELECT`** — the second query's aliases are ignored for naming.

```sql
-- Union-compatible: 2 columns, (int, text) both sides.
SELECT customer_id, name FROM customers WHERE region_id = 1
UNION
SELECT customer_id, name FROM customers WHERE region_id = 2;
```

If the column counts differ, or the types can't be reconciled, the database rejects the query outright — a helpful error, unlike the *silent* wrongness a bad join produces.

## 2. The four operators

| Operator | Returns | Duplicates |
|----------|---------|------------|
| `UNION` | rows in **either** query | removed (distinct) |
| `UNION ALL` | rows in **either** query | **kept** |
| `INTERSECT` | rows in **both** queries | removed (distinct) |
| `EXCEPT` | rows in the **first** but **not** the second | removed (distinct) |

The default for all three of `UNION`/`INTERSECT`/`EXCEPT` is **distinct** — they de-duplicate the combined result. Append `ALL` to keep duplicates (and to skip the de-dup work).

### 2.1 `UNION` and `UNION ALL`

`UNION` stacks two result sets and removes duplicate rows; `UNION ALL` stacks them and keeps every row. **`UNION ALL` is meaningfully faster** because it skips the sort/hash needed to find duplicates — so if you *know* the two sets can't overlap, or you *want* the duplicates, use `UNION ALL`.

```sql
-- Distinct list of every region_id that appears as a customer's OR a supplier's region.
SELECT region_id FROM customers WHERE region_id IS NOT NULL
UNION
SELECT region_id FROM suppliers WHERE region_id IS NOT NULL;
-- → 1, 2, 3, 4  (each once, even though several rows share a region)
```

```sql
-- Keep duplicates: every region_id occurrence across both tables.
SELECT region_id FROM customers WHERE region_id IS NOT NULL
UNION ALL
SELECT region_id FROM suppliers WHERE region_id IS NOT NULL;
-- → 1,1,1,2,2,3,  1,2,3,4  (rows preserved from both)
```

A common real use of `UNION ALL`: **label and combine heterogeneous rows** into one feed. Add a literal column that says where each row came from:

```sql
SELECT 'customer' AS party_type, name FROM customers
UNION ALL
SELECT 'supplier' AS party_type, name FROM suppliers
ORDER BY party_type, name;
```

### 2.2 `INTERSECT` — rows in both

`INTERSECT` returns only rows present in **both** result sets. "Region IDs that have **both** a customer and a supplier":

```sql
SELECT region_id FROM customers WHERE region_id IS NOT NULL
INTERSECT
SELECT region_id FROM suppliers WHERE region_id IS NOT NULL;
-- → 1, 2, 3   (region 4 has a supplier but no customer, so it's excluded)
```

### 2.3 `EXCEPT` — rows in the first, not the second

`EXCEPT` (called `MINUS` in Oracle) is set difference: rows from the first query that do **not** appear in the second. Order matters — `A EXCEPT B` ≠ `B EXCEPT A`. "Regions that have a supplier but **no** customer":

```sql
SELECT region_id FROM suppliers WHERE region_id IS NOT NULL
EXCEPT
SELECT region_id FROM customers WHERE region_id IS NOT NULL;
-- → 4   (West: has WestParts, has no customers)
```

`EXCEPT` is an anti-join expressed as sets — and often the most readable way to phrase "in this list but not that one."

### 2.4 `ALL` variants and multiset semantics

`INTERSECT ALL` and `EXCEPT ALL` exist too (in PostgreSQL) and follow **multiset** counting: if a row appears *m* times on the left and *n* times on the right, `INTERSECT ALL` keeps it `min(m,n)` times and `EXCEPT ALL` keeps it `max(m−n, 0)` times. You'll rarely need these, but know they exist.

> **SQLite:** SQLite supports `UNION`, `UNION ALL`, `INTERSECT`, and `EXCEPT`, but **not** `INTERSECT ALL` / `EXCEPT ALL`. PostgreSQL 16 supports all six. If you need the `ALL` multiset variants, you're on Postgres.

## 3. Ordering, and where `ORDER BY` goes

A set operation has **one** result, so `ORDER BY` applies to the whole thing and must come **last**, after the final `SELECT` — not inside the individual queries.

```sql
SELECT customer_id, name FROM customers WHERE region_id = 1
UNION
SELECT customer_id, name FROM customers WHERE region_id = 2
ORDER BY name;        -- sorts the combined result; refers to the FIRST query's column names
```

Because output names come from the first `SELECT`, order by those names (or by column position: `ORDER BY 2`). If you need to `ORDER BY`/`LIMIT` an *individual* branch before combining, wrap it in parentheses or a subquery — a bare `ORDER BY` inside one branch is ambiguous and often an error.

## 4. `NULL` and duplicate handling in set operations

Set operations treat two `NULL`s as **the same value** for de-duplication and matching — the opposite of `NULL = NULL` in a `WHERE`/`ON`, which is *unknown*. So:

- `UNION` collapses two `NULL` rows into one.
- `INTERSECT` matches a `NULL` on the left with a `NULL` on the right.
- `EXCEPT` removes a left `NULL` if the right side also has a `NULL`.

This "`NULL` equals `NULL`" behavior is occasionally exactly what makes `EXCEPT`/`INTERSECT` cleaner than the equivalent join, where you'd have to write `IS NOT DISTINCT FROM` to get the same matching.

## 5. Joins vs sets — choosing the right one

This is the judgment this whole week builds toward. Use the axis of *columns vs rows*:

| You want to… | Reach for |
|--------------|-----------|
| Add **columns** from a related table (name beside order) | **JOIN** |
| Stack **rows** from similar queries into one list | **UNION [ALL]** |
| Rows appearing in **both** of two lists | **INTERSECT** *(or a semi-join)* |
| Rows in one list but **not** another | **EXCEPT** *(or an anti-join)* |

The overlap is real: `INTERSECT`/`EXCEPT` and semi/anti-joins can answer the same membership questions. Two guidelines:

1. **If you only compare a key and want a set answer, the set operator is usually more readable.** "Region IDs with a supplier but no customer" is a crisp two-line `EXCEPT`; the join version needs a `LEFT JOIN` and an `IS NULL`.
2. **If you need columns from the *other* table, use a join.** `EXCEPT` can only return the columns you're comparing. The moment you want "customers who never ordered **and their signup date and region name**," you're pulling extra columns → anti-join with a `LEFT JOIN`/`NOT EXISTS`, not `EXCEPT`.

Worked comparison — "customers who never ordered," three equivalent phrasings:

```sql
-- (a) EXCEPT on the key: shortest, but only returns the id.
SELECT customer_id FROM customers
EXCEPT
SELECT customer_id FROM orders;

-- (b) Anti-join with NOT EXISTS: can return any customer columns.
SELECT c.customer_id, c.name, c.region_id
FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);

-- (c) LEFT JOIN ... IS NULL: same, join-flavored.
SELECT c.customer_id, c.name, c.region_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL;
```

All three return customer 7 (Gizmo Inc). Pick (a) when you only need the id and want the tersest query; pick (b)/(c) when the report needs more of the customer's columns.

## 6. Emulating `FULL OUTER JOIN` with `UNION` (the classic recipe)

Lecture 1 promised this. On an engine without `FULL OUTER JOIN` — old SQLite, MySQL — you rebuild it as a `LEFT JOIN` **unioned** with the right-only rows:

```sql
-- FULL OUTER JOIN customers/regions, portable version:
SELECT c.name AS customer, r.name AS region
FROM customers c
LEFT JOIN regions r ON r.region_id = c.region_id      -- all customers + matched regions
UNION
SELECT c.name AS customer, r.name AS region
FROM customers c
RIGHT JOIN regions r ON r.region_id = c.region_id     -- + regions with no customer
WHERE c.customer_id IS NULL;                          -- keep ONLY the right-only rows
```

The `UNION` de-dups the overlap; the `WHERE … IS NULL` on the second branch adds only the rows the `LEFT JOIN` couldn't reach. On engines without `RIGHT JOIN` either, swap the tables in the second branch. This recipe is worth recognizing because you'll meet it in older codebases.

## 7. Check yourself

- What are the three union-compatibility rules? Where do the result's column names come from?
- `UNION` vs `UNION ALL` — which is faster and why? When must you use `ALL`?
- Predict the output of `{customers' regions} EXCEPT {suppliers' regions}` in `crunch_shop`.
- Where does `ORDER BY` go in a set operation, and which query's column names does it use?
- How do set operators treat `NULL = NULL` differently from a `WHERE` clause?
- Give one question best answered by `EXCEPT` and one best answered by an anti-join that returns extra columns.
- Sketch the `UNION` recipe that emulates a `FULL OUTER JOIN`.

When those are automatic, you've finished the lectures. Do the remaining exercises, then the [mini-project](../mini-project/README.md).

## Further reading

- **PostgreSQL — combining queries (`UNION`/`INTERSECT`/`EXCEPT`):** <https://www.postgresql.org/docs/16/queries-union.html>
- **SQLite — compound select statements:** <https://www.sqlite.org/lang_select.html#compound_select_statements>
- **SQL set operations, visually (Modern SQL, Markus Winand):** <https://modern-sql.com/caniuse/intersect>
