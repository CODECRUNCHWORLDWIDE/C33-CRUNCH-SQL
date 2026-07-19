# Week 5 — Resources

Free and official first, then the writing worth your time. Everything here is public; a few books are paid but their linked chapters or summaries are free.

## Install / setup

- **PostgreSQL 16 downloads** — official installers for macOS, Linux, Windows: <https://www.postgresql.org/download/>
  - *Why:* you need a local PG 16 to run every exercise this week.
- **`psql` — the interactive terminal:** <https://www.postgresql.org/docs/16/app-psql.html>
  - *Why:* the whole week lives in `psql`; skim the meta-commands (`\d`, `\x`, `\watch`).
- **SQLite download / CLI:** <https://www.sqlite.org/cli.html>
  - *Why:* the contrast engine — one-writer serializability, used in Lectures 1–3's asides.

## The official documentation (read these, in order)

- **PostgreSQL 16 — Transaction Isolation:** <https://www.postgresql.org/docs/16/transaction-iso.html>
  - *Why:* the single most important page this week — the levels, exactly which anomalies each prevents, and worked write-skew examples. Read it twice.
- **PostgreSQL 16 — Concurrency Control (MVCC intro):** <https://www.postgresql.org/docs/16/mvcc-intro.html>
  - *Why:* the "readers don't block writers" model in the project's own words.
- **PostgreSQL 16 — Explicit Locking:** <https://www.postgresql.org/docs/16/explicit-locking.html>
  - *Why:* the full table of row and table lock modes and what conflicts with what — the reference behind Lecture 3 §5.
- **PostgreSQL 16 — `SAVEPOINT`, `ROLLBACK TO`, `RELEASE`:** <https://www.postgresql.org/docs/16/sql-savepoint.html>
  - *Why:* exact syntax and nesting rules for Exercise 1.
- **PostgreSQL 16 — Routine Vacuuming:** <https://www.postgresql.org/docs/16/routine-vacuuming.html>
  - *Why:* why MVCC needs vacuum, plus bloat and transaction-id wraparound explained by the maintainers.
- **SQLite — Isolation & file locking:** <https://www.sqlite.org/isolation.html> and <https://www.sqlite.org/lockingv3.html>
  - *Why:* how a database delivers serializability by allowing one writer at a time.
- **SQLite — Write-Ahead Logging (WAL):** <https://www.sqlite.org/wal.html>
  - *Why:* how SQLite lets readers proceed during a write — a slice of what MVCC does.

## Papers and deep dives

- **Berenson, Bernstein, Gray, et al. — "A Critique of ANSI SQL Isolation Levels" (1995):** <https://www.microsoft.com/en-us/research/publication/a-critique-of-ansi-sql-isolation-levels/>
  - *Why:* shows the standard's anomaly definitions are ambiguous and introduces snapshot isolation and write skew — the intellectual root of this whole week.
- **Ports & Grittner — "Serializable Snapshot Isolation in PostgreSQL" (2012):** <https://drkp.net/papers/ssi-vldb12.pdf>
  - *Why:* how Postgres's SERIALIZABLE (SSI) actually detects the dangerous read-write cycles it aborts with `40001`.
- **PostgreSQL wiki — SSI and the retry pattern:** <https://wiki.postgresql.org/wiki/SSI>
  - *Why:* practical guidance and the canonical retry-loop advice for serialization failures.

## Books (chapters worth the money, summaries free)

- **Martin Kleppmann — *Designing Data-Intensive Applications*, Chapter 7 ("Transactions"):**
  - *Why:* the clearest modern explanation of weak isolation, lost updates, and write skew anywhere. If you buy one database book, this is it.
- **"The Art of PostgreSQL" — Dimitri Fontaine** (site has free excerpts): <https://theartofpostgresql.com/>
  - *Why:* practitioner-focused PG, including transactional patterns.

## Posts and talks (free)

- **"PostgreSQL rocks, except when it blocks" — Marco Slot / Citus blog:** search the Citus Data blog for locking-and-blocking write-ups.
  - *Why:* concrete stories of `ALTER TABLE` and lock queues freezing a production app — Lecture 3 §5 made real.
- **Brandur Leach — "How Postgres makes transactions atomic":** <https://brandur.org/postgres-atomicity>
  - *Why:* a careful, readable walk through the WAL and commit path behind atomicity and durability.
- **Bruce Momjian — "MVCC Unmasked" (slides/talk):** <https://momjian.us/main/presentations/internals.html>
  - *Why:* a core PG contributor's visual tour of tuples, `xmin`/`xmax`, and vacuum.

## Cheat card — the numbers to memorize

| Thing | Value |
|-------|-------|
| Default isolation level (Postgres) | `READ COMMITTED` |
| Distinct isolation levels Postgres really has | 3 (READ UNCOMMITTED aliases to READ COMMITTED) |
| Serialization-failure SQLSTATE | `40001` |
| Deadlock SQLSTATE | `40P01` |
| Deadlock detection delay | `deadlock_timeout`, default **1 s** |
| Anomaly left at Postgres REPEATABLE READ | serialization anomaly (write skew) only |
| Row-lock clause that prevents lost updates | `SELECT … FOR UPDATE` |

---

*Broken link? Open an issue or PR.*
