# Lecture 2 — `SELECT`, Deep

> **Duration:** ~2 hours. **Outcome:** You can project any columns or expressions with aliases, filter rows with the full `WHERE` toolkit (comparisons, `AND`/`OR`/`NOT`, `IN`, `BETWEEN`, `LIKE`/`ILIKE`, `IS NULL`), sort with `ORDER BY`, remove duplicates with `DISTINCT`, and page results with `LIMIT`/`OFFSET` — writing all of it from memory.

`SELECT` is the verb you'll type more than any other for the rest of your career. This lecture takes it apart clause by clause. Run every example against the seed table as you read — reading SQL and writing SQL are different skills, and only one of them gets you a job.

## 1. Projection — choosing columns

The list right after `SELECT` is the **projection**: which columns come back, in what order.

```sql
SELECT first_name, last_name, salary FROM employees;
SELECT * FROM employees;              -- every column; fine when exploring
```

`SELECT *` is convenient for poking around but bad in real code — it pulls columns you don't need (slower, more network) and breaks silently when someone adds a column. **Name your columns** in anything you'll keep.

### Expressions and computed columns

The projection list isn't limited to bare columns. You can compute:

```sql
SELECT
    first_name,
    salary,
    salary * 1.10                 AS salary_after_raise,
    salary / 12                   AS monthly_pay,
    first_name || ' ' || last_name AS full_name   -- || is string concatenation
FROM employees;
```

`||` is the SQL-standard concatenation operator (works on both Postgres and SQLite). A computed column has an ugly auto-generated name unless you rename it — which is what `AS` is for.

### Aliases with `AS`

An **alias** renames a column (or, later, a table) in the output:

```sql
SELECT salary AS annual_salary,
       salary / 12 AS monthly   -- AS is optional but read-clearer
FROM employees;
```

Rules worth knowing:

- `AS` is optional (`salary annual_salary` works) but include it — it reads better and prevents mistakes.
- If your alias has spaces or capitals you want preserved, double-quote it: `AS "Monthly Pay"`. Single quotes are for **string values**, double quotes are for **identifiers**. Mixing these up is the #1 beginner syntax error.
- You **cannot** use a `SELECT` alias in the `WHERE` clause (see the evaluation-order note in Lecture 1) — but you *can* use it in `ORDER BY`, because `ORDER BY` runs last.

## 2. `WHERE` — choosing rows

`WHERE` keeps only rows for which its condition is **true**. (Not false, and — critically — not `NULL`/unknown. Hold that thought for Lecture 3.)

### Comparison operators

```sql
SELECT * FROM employees WHERE salary > 150000;
SELECT * FROM employees WHERE department = 'Sales';     -- = is equality, not ==
SELECT * FROM employees WHERE salary <> 100000;         -- <> is "not equal" (!= also works)
SELECT * FROM employees WHERE hire_date >= '2020-01-01';
```

| Operator | Meaning |
|----------|---------|
| `=` | equal (single `=`, not `==`) |
| `<>` or `!=` | not equal |
| `<` `<=` `>` `>=` | ordering comparisons (work on numbers, text, dates) |

String comparison is case-**sensitive** and lexicographic: `'Sales' = 'sales'` is **false**. Dates compare chronologically when written `'YYYY-MM-DD'`.

### Combining conditions: `AND`, `OR`, `NOT`

```sql
-- Engineers who are remote
SELECT * FROM employees WHERE department = 'Engineering' AND is_remote = TRUE;

-- Anyone in Sales OR Marketing
SELECT * FROM employees WHERE department = 'Sales' OR department = 'Marketing';

-- Everyone NOT in Engineering
SELECT * FROM employees WHERE NOT department = 'Engineering';
```

**Precedence matters.** `NOT` binds tightest, then `AND`, then `OR`. So this:

```sql
WHERE department = 'Sales' OR department = 'Marketing' AND salary > 100000
```

means `Sales OR (Marketing AND >100k)` — probably **not** what you meant. Use parentheses to say exactly what you mean, every time:

```sql
WHERE (department = 'Sales' OR department = 'Marketing') AND salary > 100000
```

Parentheses cost nothing and prevent a whole category of silent bugs. Use them liberally.

### `IN` — membership in a list

`IN` is cleaner than a chain of `OR`s:

```sql
-- these two are equivalent
SELECT * FROM employees WHERE department = 'Sales' OR department = 'Marketing' OR department = 'HR';
SELECT * FROM employees WHERE department IN ('Sales', 'Marketing', 'HR');

SELECT * FROM employees WHERE department NOT IN ('Engineering', 'Sales');
```

⚠️ **`NOT IN` and `NULL` don't mix.** If the list (or the column) contains a `NULL`, `NOT IN` can return *zero rows* unexpectedly. We dissect exactly why in Lecture 3 — for now, be wary of `NOT IN` on nullable columns.

### `BETWEEN` — inclusive ranges

```sql
-- salary from 80k to 120k, INCLUSIVE of both ends
SELECT * FROM employees WHERE salary BETWEEN 80000 AND 120000;

-- hired in 2021 (dates work too)
SELECT * FROM employees WHERE hire_date BETWEEN '2021-01-01' AND '2021-12-31';
```

`BETWEEN a AND b` means `>= a AND <= b` — **both endpoints are included**. That inclusivity is the classic gotcha; if you want an exclusive upper bound, write the explicit `>=`/`<` form instead.

### `LIKE` and `ILIKE` — pattern matching

For "starts with," "contains," "ends with" on text, use `LIKE` with two wildcards:

- `%` matches **any sequence** of characters (including none).
- `_` matches **exactly one** character.

```sql
SELECT * FROM employees WHERE last_name LIKE 'S%';       -- surname starts with S
SELECT * FROM employees WHERE email LIKE '%@crunch.io';  -- ends with the domain
SELECT * FROM employees WHERE first_name LIKE '%a%';     -- contains an 'a'
SELECT * FROM employees WHERE first_name LIKE '_a%';     -- 2nd letter is 'a'
```

`LIKE` is **case-sensitive in PostgreSQL**. To match case-insensitively in Postgres, use **`ILIKE`**:

```sql
-- Postgres: matches 'smith', 'Smith', 'SMITH'
SELECT * FROM employees WHERE last_name ILIKE 's%';
```

**Engine difference:** SQLite has no `ILIKE`; its plain `LIKE` is *already* case-insensitive for ASCII letters (`LIKE 's%'` matches `Smith`). So on SQLite you just use `LIKE`; on Postgres you choose `LIKE` (exact case) or `ILIKE` (ignore case). To match a literal `%` or `_`, use an `ESCAPE` clause: `LIKE '100\%' ESCAPE '\'`.

### `IS NULL` — the only way to test for missing values

`NULL` means "unknown / no value." You **cannot** test it with `=`. `email = NULL` is never true — it's `NULL`. The correct test:

```sql
SELECT * FROM employees WHERE email IS NULL;       -- the two with no email
SELECT * FROM employees WHERE email IS NOT NULL;   -- everyone else
SELECT * FROM employees WHERE commission_pct IS NULL;  -- all non-Sales staff
```

Burn this in: **`IS NULL` / `IS NOT NULL`, never `= NULL`.** Lecture 3 explains the three-valued logic that makes this necessary.

## 3. `ORDER BY` — sorting the result

Rows come back in no guaranteed order unless you ask. `ORDER BY` sorts the final result:

```sql
SELECT first_name, salary FROM employees ORDER BY salary DESC;   -- highest first
SELECT first_name, salary FROM employees ORDER BY salary ASC;    -- lowest first (ASC is the default)
```

**Multi-key sorting** — sort by the first column, break ties with the next:

```sql
SELECT department, last_name, salary
FROM employees
ORDER BY department ASC, salary DESC;   -- group by dept, and within each, highest paid first
```

You can order by an alias, by a computed expression, or even by output-column position (`ORDER BY 3` = the third selected column — handy interactively, avoid in saved code):

```sql
SELECT first_name, last_name, salary * 12 AS yearly FROM employees ORDER BY yearly DESC;
```

**Where do `NULL`s sort?** By default: Postgres puts `NULL`s **last** for `ASC` and **first** for `DESC`; SQLite puts `NULL`s **first** for `ASC`. Don't rely on the default — say it explicitly with `NULLS FIRST` / `NULLS LAST` (Postgres) when it matters:

```sql
SELECT first_name, commission_pct
FROM employees
ORDER BY commission_pct DESC NULLS LAST;   -- Postgres: real commissions first, NULLs at the bottom
```

## 4. `DISTINCT` — removing duplicate rows

`DISTINCT` collapses duplicate rows in the result to one each:

```sql
SELECT DISTINCT department FROM employees;              -- the 8 unique departments
SELECT DISTINCT department, is_remote FROM employees;   -- unique (dept, remote) COMBINATIONS
```

Two things people get wrong:

1. `DISTINCT` applies to the **whole row of the projection**, not just the first column. `SELECT DISTINCT department, city` gives distinct *pairs*, not distinct departments.
2. `NULL`s are treated as equal to each other for `DISTINCT` — all the `NULL` emails collapse to a single `NULL` in `SELECT DISTINCT email`.

## 5. `LIMIT` and `OFFSET` — paging

`LIMIT n` returns at most `n` rows; `OFFSET m` skips the first `m` first:

```sql
SELECT first_name, salary FROM employees ORDER BY salary DESC LIMIT 5;            -- top 5 earners
SELECT first_name, salary FROM employees ORDER BY salary DESC LIMIT 5 OFFSET 5;   -- the NEXT 5 (rows 6–10)
```

That second query is **page 2** of a "5 per page" listing. Both Postgres and SQLite support `LIMIT`/`OFFSET` (the SQL-standard spelling is `FETCH FIRST n ROWS ONLY`, but `LIMIT` is universal in these two engines).

**Always pair `LIMIT` with `ORDER BY`.** Without an `ORDER BY`, "the top 5" is meaningless — remember, a table is an unordered set, so `LIMIT 5` alone returns *some* 5 rows, and which 5 can change between runs. A `LIMIT` without an `ORDER BY` is almost always a bug.

## 6. Putting the clauses together

The canonical query shape, written order vs. the order the engine evaluates them:

| Written order | Evaluation order | Job |
|---------------|------------------|-----|
| `SELECT` | 4 | pick/compute output columns |
| `FROM` | 1 | choose the source table |
| `WHERE` | 2 | keep matching rows |
| `ORDER BY` | 5 | sort the result |
| `LIMIT`/`OFFSET` | 6 | take a slice |

(`GROUP BY`/`HAVING` slot in at step 3; that's Week 3.) The mismatch is why an alias defined in `SELECT` is usable in `ORDER BY` (step 5, after `SELECT`) but not in `WHERE` (step 2, before `SELECT`).

A full example — "the 3 highest-paid remote employees outside Engineering, newest hires breaking ties":

```sql
SELECT first_name || ' ' || last_name AS name,
       department,
       salary
FROM employees
WHERE is_remote = TRUE
  AND department <> 'Engineering'
ORDER BY salary DESC, hire_date DESC
LIMIT 3;
```

Read it top to bottom in *evaluation* order and you can see exactly what the engine does.

## 7. Check yourself

- What's the difference between single quotes and double quotes in SQL?
- Why should you avoid `SELECT *` in code you'll keep?
- Rewrite `dept = 'A' OR dept = 'B' OR dept = 'C'` using `IN`.
- Is `BETWEEN 80000 AND 120000` inclusive or exclusive of the endpoints?
- Write the pattern for "email ends in `@crunch.io`". Which wildcard did you use?
- Why is `WHERE email = NULL` always wrong, and what do you write instead?
- On PostgreSQL, how do you match a name case-insensitively? What about on SQLite?
- Why is a `LIMIT` without an `ORDER BY` almost always a bug?
- Which clause can use a `SELECT` alias — `WHERE` or `ORDER BY` — and why?

If those are automatic, Lecture 3 goes under the hood on data types and the `NULL` logic that quietly breaks queries.

## Further reading

- **PostgreSQL — "Queries" (SELECT):** <https://www.postgresql.org/docs/current/queries.html>
- **PostgreSQL — `SELECT` reference:** <https://www.postgresql.org/docs/current/sql-select.html>
- **SQLite — `SELECT` syntax:** <https://www.sqlite.org/lang_select.html>
- **PostgreSQL — Pattern matching (`LIKE`/`ILIKE`/regex):** <https://www.postgresql.org/docs/current/functions-matching.html>
