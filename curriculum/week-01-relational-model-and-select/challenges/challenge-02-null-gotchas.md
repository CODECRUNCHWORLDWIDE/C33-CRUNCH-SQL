# Challenge 2 — NULL Gotchas

**Time:** ~60 minutes. **Difficulty:** Medium. **No single right answer** (but there are definitely *wrong* ones).

## The scenario

A teammate hands you six queries. Each one **runs without error** and each one is **subtly wrong** — it returns the wrong rows, or the wrong count, because of `NULL` and three-valued logic. This is the most common category of real SQL bug, and the most invisible: nothing crashes, the number just isn't right.

Your job: for each query, (1) predict what it returns, (2) explain *why* it's wrong, (3) write the corrected version, and (4) confirm the fix against the data. Put your work in `challenge-02.md`.

Reminder from Lecture 3: `WHERE` keeps a row only when the condition is **true** — rows where it's **false** *or* **unknown** are dropped. Every bug below is a "the condition came out `unknown`" bug.

## The six suspect queries

**Bug 1 — the disappearing rows.**
```sql
-- "Everyone who does NOT earn a 12% commission"
SELECT first_name FROM employees WHERE commission_pct <> 0.12;
```
The team expects 28 rows (30 minus the two 12%-ers). It returns far fewer. Why? Fix it so all 28 non-12% people show up — including the ones with no commission at all.

**Bug 2 — `NOT IN` returns nothing.**
```sql
-- "Everyone whose manager is NOT 9, 12, or the CEO's (NULL) manager"
SELECT first_name FROM employees WHERE manager_id NOT IN (9, 12, NULL);
```
This returns **zero rows**. Explain the expansion that makes every row fail, then rewrite it to mean what was intended.

**Bug 3 — the average that's too high.**
```sql
-- "Average commission percentage across the company"
SELECT AVG(commission_pct) FROM employees;
```
Someone reports this as "the company's average commission." Why is that claim misleading? What is this number actually the average *of*? Write the two different queries for "average across all 30 employees (treating no-commission as 0)" versus "average among those who earn commission," and explain when each is the right question.

**Bug 4 — the empty-string confusion.**
```sql
-- "Employees with no email on file"
SELECT first_name FROM employees WHERE email = '';
```
This returns nothing, yet two employees truly have no email. What's the difference between an **empty string** and a **`NULL`**, and what's the correct predicate?

**Bug 5 — the negation that leaks.**
```sql
-- "Everyone NOT based in Miami"
SELECT first_name, city FROM employees WHERE city <> 'Miami';
```
In our seed every `city` happens to be filled in, so this looks fine. But suppose HR adds a new hire whose `city` is still `NULL`. Predict what happens to that row. Rewrite the query so a `NULL`-city employee is correctly included in "not based in Miami," and explain your reasoning.

**Bug 6 — `DISTINCT` and `NULL`.**
```sql
-- "How many distinct email values are there?"
SELECT COUNT(DISTINCT email) FROM employees;
SELECT COUNT(*) FROM employees;
```
The two counts differ by more than the two missing emails. Explain how `COUNT(DISTINCT ...)` and `COUNT(*)` each treat `NULL`, and state exactly what number each query returns and why.

## Constraints

- For every bug, your write-up must contain **all four** parts: prediction, explanation, corrected query, confirmation.
- "It just works now" is not an explanation. Name the three-valued-logic rule at play (e.g., "`x <> NULL` evaluates to `unknown`, and `WHERE` drops `unknown` rows").
- You may use `COALESCE`, `IS [NOT] NULL`, `IS [NOT] DISTINCT FROM`, and reshaping the predicate. Pick the clearest fix, not the cleverest.

## How success is judged

| Signal | Weak | Strong |
|--------|------|--------|
| Diagnosis | "It returns the wrong thing" | Names the exact `unknown`-producing comparison |
| Fix | Patches symptoms | Predicate now expresses the real intent |
| `NULL` model | Treats `NULL` as 0 or `''` | Distinguishes unknown / not-applicable / empty |
| Aggregation insight (Bug 3) | One number, no nuance | Two queries + when each is correct |
| Confirmation | None | States the row count/value before and after |

## Stretch

- Write a single query that lists, for every employee, their `commission_pct` **and** a second column that shows `0` where the commission is `NULL` — proving you can present "missing" and "zero" side by side without conflating them.
- Construct a `WHERE` clause that returns **exactly** the two `NULL`-email employees **and** anyone whose email doesn't end in `@crunch.io`, in one predicate. (There are none of the latter in the seed — so this should return exactly 2. Prove it.)

## Submission

Commit `challenge-02.md` to your portfolio under `c33-week-01/challenge-02/`.
