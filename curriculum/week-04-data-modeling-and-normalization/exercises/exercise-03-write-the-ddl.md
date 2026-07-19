# Exercise 3 — Write the DDL

**Goal:** Translate a normalized model into runnable `CREATE TABLE` statements carrying the full constraint set — PK, FK with the right `ON DELETE` action, NOT NULL, UNIQUE, CHECK — and then *prove* each constraint works by trying to violate it and watching the database reject you.

**Estimated time:** 50 minutes. Use **PostgreSQL 16** if you can (SQLite works but needs `PRAGMA foreign_keys = ON;` and `STRICT` tables to enforce everything).

## The model

Build the library schema you designed in Exercise 1. Concretely, create these tables:

- `members(member_id PK, email UNIQUE NOT NULL, full_name NOT NULL, joined_on DATE default today)`
- `books(book_id PK, isbn UNIQUE NOT NULL, title NOT NULL, published_year with a sane range CHECK)`
- `copies(copy_id PK, book_id FK→books, barcode UNIQUE NOT NULL, shelf NOT NULL)` — deleting a book should delete its copies.
- `borrowings(borrowing_id PK, member_id FK→members, copy_id FK→copies, borrowed_on NOT NULL default today, returned_on nullable)` — deleting a member should be **blocked** if they have borrowings; a CHECK must ensure `returned_on >= borrowed_on` when present.

## Your task

1. **Write the DDL** into `schema.sql`, in dependency order (parents before children). Use `BIGINT GENERATED ALWAYS AS IDENTITY` for surrogate PKs in Postgres (`INTEGER PRIMARY KEY` in SQLite).
2. **Run it** into a fresh database. Fix every error until it creates cleanly.
3. **Seed a little data** — two members, two books, three copies, two borrowings — so you have rows to test against.
4. **Break it on purpose.** Write, in `proofs.sql`, one statement per constraint that *should fail*, run each, and paste the error message. You must demonstrate at least these six rejections:

   | # | Try to… | Constraint that should stop you |
   |---|---------|--------------------------------|
   | 1 | insert two members with the same email | `UNIQUE` |
   | 2 | insert a copy whose `book_id` doesn't exist | `FOREIGN KEY` |
   | 3 | insert a book with `published_year = 3000` | `CHECK` |
   | 4 | insert a member with `full_name = NULL` | `NOT NULL` |
   | 5 | delete a member who has an open borrowing | `ON DELETE RESTRICT`/`NO ACTION` |
   | 6 | insert a borrowing with `returned_on` before `borrowed_on` | multi-column `CHECK` |

5. **Prove CASCADE works** — delete a `book` that has copies and show the copies vanished (this one *should* succeed). Paste the before/after `SELECT count(*)`.

## Expected result

A `schema.sql` that builds cleanly and a `proofs.sql` where six statements are rejected with clear errors and one cascade succeeds. Example of a "working" rejection in Postgres:

```
ERROR:  new row for relation "books" violates check constraint "books_published_year_check"
DETAIL:  Failing row contains (3, 978..., Some Title, 3000).
```

That error is the constraint doing its job. A schema where these inserts *succeed* is a schema that isn't protecting your data.

## Done when…

- [ ] `schema.sql` creates all four tables cleanly, parents before children.
- [ ] Every FK declares an explicit `ON DELETE` action (`CASCADE` for copies, `RESTRICT`/`NO ACTION` for borrowings).
- [ ] `proofs.sql` shows all **six** rejections with their actual error text pasted beneath.
- [ ] The CASCADE demonstration shows copies disappearing when their book is deleted.
- [ ] (SQLite users) You ran `PRAGMA foreign_keys = ON;` and used `STRICT` tables — otherwise proofs 2 and 5 won't fire.

## Stretch

- Add the "one active loan per copy" rule from Exercise 1's stretch. In Postgres, a **partial unique index** does it: `CREATE UNIQUE INDEX one_active_loan ON borrowings(copy_id) WHERE returned_on IS NULL;`. Insert a second open borrowing on a copy that's already out and confirm it's rejected.
- Add a `fines` table linked to `borrowings`, with a CHECK that `amount >= 0`, and wire its `ON DELETE` sensibly.
- Re-run your whole `schema.sql` a second time. It errors ("relation already exists"). Add `DROP TABLE IF EXISTS ... CASCADE;` statements at the top *in reverse dependency order* so the script is idempotent — a habit every real migration needs.

## Submission

Commit `schema.sql` and `proofs.sql` to `c33-week-04/exercise-03/`.
