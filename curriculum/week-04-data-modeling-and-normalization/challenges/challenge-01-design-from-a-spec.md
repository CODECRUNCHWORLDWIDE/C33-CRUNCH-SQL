# Challenge 1 — Design a Schema From a Spec

**Type:** open-ended design. No single correct answer — defend your choices.

**Estimated time:** 2–3 hours.

## The scenario

A small clinic hires you to design the database behind its appointment system. They hand you this description — the kind of vague, real-world prose you'll get your whole career:

> Patients book appointments with doctors. A patient has contact details and one or more insurance policies (each policy has a provider, a policy number, and a coverage-start date). A doctor works in one or more specialties (cardiology, pediatrics, …) and practises at one or more of the clinic's locations. An appointment is between one patient and one doctor, at one location, at a specific date and time, and has a status (booked, completed, cancelled, no-show). During a completed appointment the doctor may record several diagnoses (each with a standard ICD-10 code and free-text notes) and may prescribe several medications (each with a drug name, dosage, and duration). The clinic must never double-book a doctor into two appointments at the same time, and must be able to list, for any patient, every medication they've ever been prescribed and by whom.

## Your task

Deliver three artifacts.

1. **An ER diagram** (`schema.md` with Mermaid, or an image) showing every entity, its keys, and every relationship with cardinality. Expect roughly 8–12 tables once you resolve the many-to-manys — there are several (doctor↔specialty, doctor↔location, and the appointment's diagnoses and prescriptions).
2. **Runnable DDL** (`schema.sql`) — `CREATE TABLE`s in PostgreSQL 16 with full constraints: surrogate PKs, FKs with deliberate `ON DELETE` actions, NOT NULL where a fact is mandatory, UNIQUE for natural keys, and CHECK for every enumerable or bounded value (status, dates, etc.).
3. **A design memo** (`decisions.md`, ~1 page) defending your non-obvious calls.

## Constraints you must satisfy

- **At least 3NF** across the board. Any deliberate denormalization must be flagged and justified in the memo.
- The **"never double-book a doctor"** rule must be enforced by the *schema*, not the app. (Hint: a `UNIQUE (doctor_id, starts_at)` constraint is the simple version; a Postgres `EXCLUDE` constraint with a time range is the senior version — either is acceptable if you explain it.)
- Every many-to-many is resolved with a junction table.
- The "list every medication a patient was ever prescribed, and by whom" query must be answerable — meaning prescriptions must link back through the appointment to both patient and doctor. Include that `SELECT` in `decisions.md` to prove your model supports it.

## Hints

<details>
<summary>Where the tricky modeling is</summary>

- **Insurance policies** are a 1:N off patient (a patient has many), *not* columns on the patient row. Repeating `policy1_provider`, `policy2_provider` is the mistake to avoid.
- **Diagnoses and prescriptions** belong to a specific *appointment*, not to the patient in general — that's how you get "by whom" for free (the appointment knows the doctor). Hang them off `appointments`.
- **Specialties and locations** are both M:N with doctors → two junction tables.
- The **status** field is a textbook `CHECK (status IN (...))`.
- ICD-10 codes are a natural key for a `diagnoses`/`icd_codes` lookup table — decide whether you want a reference table of valid codes or just store the code.

</details>

## How success is judged

| Dimension | What "good" looks like |
|-----------|------------------------|
| Completeness | Every fact in the spec has a home; nothing is unstorable. |
| Normalization | 3NF+ ; no repeating groups, no partial/transitive dependencies. |
| Integrity | Double-booking is *impossible*; orphans are *impossible*; statuses are bounded. |
| Junctions | Every M:N resolved; relationship attributes placed correctly. |
| Defensibility | The memo explains keys, `ON DELETE` choices, and any denormalization. |

## Deliverable

Commit `schema.md`, `schema.sql`, and `decisions.md` to `c33-week-04/challenge-01/`. Your `schema.sql` must run cleanly into an empty PostgreSQL 16 database (`psql -f schema.sql`).
