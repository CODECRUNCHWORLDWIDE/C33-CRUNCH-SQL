# Week 4 — Resources

Free, public, no signup unless noted. Skim the required reading before the lectures; keep the references open while you work.

## Required reading

- **PostgreSQL 16 — `CREATE TABLE`** — the authoritative reference for every constraint syntax you'll use this week: <https://www.postgresql.org/docs/16/sql-createtable.html>
  *Why:* it's the exact grammar for PK, FK, UNIQUE, CHECK, and `ON DELETE` — bookmark it.
- **PostgreSQL 16 — Constraints (tutorial chapter)** — a readable walkthrough of each constraint type with examples: <https://www.postgresql.org/docs/16/ddl-constraints.html>
  *Why:* the gentlest correct explanation of foreign keys and referential actions anywhere.
- **PostgreSQL 16 — Data Types** — pick the right column type instead of defaulting to `TEXT`: <https://www.postgresql.org/docs/16/datatype.html>
  *Why:* the type is your cheapest constraint; this is the menu.

## Normalization theory

- **Wikipedia — Database normalization** — solid, example-driven overview of 1NF–5NF and the anomalies: <https://en.wikipedia.org/wiki/Database_normalization>
  *Why:* the anomaly examples are clear and the historical context (Codd) is worth having.
- **Wikipedia — Functional dependency** — the formal tool underneath every normal form: <https://en.wikipedia.org/wiki/Functional_dependency>
  *Why:* if FDs click, normalization stops being memorization.
- **Wikipedia — Boyce–Codd normal form** — the overlapping-key edge case, worked: <https://en.wikipedia.org/wiki/Boyce%E2%80%93Codd_normal_form>
  *Why:* the one place 3NF and BCNF diverge, explained with the classic example.

## ER modeling

- **Mermaid — Entity Relationship Diagrams** — text-to-diagram syntax that renders on GitHub: <https://mermaid.js.org/syntax/entityRelationshipDiagram.html>
  *Why:* version-controllable ER diagrams; what the lectures and exercises use.
- **Excalidraw** — free, browser-based, sketchy-aesthetic whiteboard for drawing schemas by hand: <https://excalidraw.com/>
  *Why:* fastest way to sketch an ERD when you're thinking, not documenting.
- **dbdiagram.io** — write DBML, get an ER diagram + exportable SQL (free tier, no signup to try): <https://dbdiagram.io/>
  *Why:* great for quickly visualizing a schema and eyeballing the relationships.

## Reference & practice

- **PostgreSQL 16 — `ALTER TABLE`** — how to evolve a live schema: <https://www.postgresql.org/docs/16/sql-altertable.html>
  *Why:* real schemas change; this is the surgery manual.
- **SQLite — Foreign Key Support** — the `PRAGMA foreign_keys = ON;` gotcha, in detail: <https://www.sqlite.org/foreignkeys.html>
  *Why:* explains exactly why SQLite silently allows orphans by default.
- **SQLite — STRICT tables** — how to make SQLite actually enforce column types: <https://www.sqlite.org/stricttables.html>
  *Why:* use this when practicing modeling in SQLite so types bite like they do in Postgres.
- **PostgreSQL — Don't Do This (wiki)** — a curated list of schema anti-patterns straight from the community: <https://wiki.postgresql.org/wiki/Don%27t_Do_This>
  *Why:* `SERIAL` vs. identity, `timestamp` vs. `timestamptz`, `char(n)` — the exact traps this week warns about.

## Deeper (optional)

- **"A Relational Model of Data for Large Shared Data Banks" — E. F. Codd (1970)** — the original paper that started all of this: <https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf>
  *Why:* short, historic, and clarifies *why* the relational model exists. Read it once.
- **use The Index, Luke — "Anatomy of a table"** — free web book; a preview of how your schema meets indexing (Week 6): <https://use-the-index-luke.com/>
  *Why:* your modeling decisions this week become performance decisions next month.

## Glossary

| Term | Definition |
|------|------------|
| **Entity** | A thing you store data about; becomes a table. |
| **Attribute** | A property of an entity; becomes a column. |
| **Primary key (PK)** | The chosen candidate key; the row's identity. NOT NULL + UNIQUE. |
| **Candidate key** | A minimal uniquely-identifying set of columns. |
| **Foreign key (FK)** | A column referencing another table's PK; enforces referential integrity. |
| **Surrogate key** | A meaningless generated id (identity/UUID) used as PK. |
| **Natural key** | A real-world unique attribute (email, ISBN) usable as a key. |
| **Junction table** | A table resolving an M:N into two 1:N relationships. |
| **Functional dependency** | `X → Y`: same X always implies same Y. |
| **Anomaly** | An update/insert/delete failure caused by redundancy. |
| **Normalization** | Reorganizing tables so each fact is stored once. |
| **3NF** | No non-key column depends on another non-key column. |
| **BCNF** | Every determinant of a functional dependency is a superkey. |
| **Referential action** | What `ON DELETE`/`ON UPDATE` does to children (CASCADE, RESTRICT, SET NULL…). |
| **Denormalization** | Deliberately reintroducing redundancy for read performance. |

---

*Broken link? Open an issue or PR.*
