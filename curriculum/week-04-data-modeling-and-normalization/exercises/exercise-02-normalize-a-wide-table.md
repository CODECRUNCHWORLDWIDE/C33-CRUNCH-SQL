# Exercise 2 — Normalize a Wide Table

**Goal:** Take one deliberately messy, wide table and walk it up the normalization ladder — 1NF → 2NF → 3NF — writing down the functional dependency you're removing and the anomaly it causes at every step. This is the reasoning drill; you'll build the final DDL in Exercise 3.

**Estimated time:** 45 minutes.

## The messy table

A conference tracks talk submissions in one spreadsheet-turned-table, `submissions`:

| sub_id | speaker | speaker_email | speaker_company | company_city | talk_titles | track | track_room |
|-------:|---------|---------------|-----------------|--------------|-------------|-------|------------|
| 1 | A. Rivera | ariv@x.io | Nimbus | Austin | "Scaling PG; Index deep-dive" | Data | Hall A |
| 2 | B. Osei | bosei@y.io | Delta | Denver | "RLS in practice" | Security | Hall B |
| 3 | A. Rivera | ariv@x.io | Nimbus | Austin | "Vacuum tuning" | Data | Hall A |
| 4 | C. Lin | clin@z.io | Nimbus | Austin | "JSONB patterns; GIN indexes" | Data | Hall A |

Assume these business rules:

- A speaker has one email and works at one company; `speaker_email → speaker`, `speaker → speaker_company`.
- A company is in one city; `speaker_company → company_city`.
- A track is held in one room; `track → track_room`.
- `talk_titles` packs *all* of a submission's talks into one cell, separated by `;`.
- `sub_id` is the key.

## Your task

In `normalize.md`, produce four sections. In each, show the table(s), list the functional dependencies, and name the specific anomaly you are removing.

### Step 0 — Identify all functional dependencies

Before touching anything, list every FD you can find in the wide table. (There are at least five.) This is the raw material for every step that follows.

### Step 1 — Reach 1NF

`talk_titles` violates 1NF (a repeating group crammed into one cell). Split it out. State: which rule of 1NF is violated, and what querying problem it causes *before* you fix it (e.g., "count talks in the Data track"). Show the resulting tables and choose the new key(s).

### Step 2 — Reach 2NF

After 1NF you'll have a talks table with a composite key. Find any **partial dependency** (a non-key column depending on only part of the key). Decompose. Name the partial FD and the redundancy it caused.

### Step 3 — Reach 3NF

Hunt for **transitive dependencies** — non-key → non-key. There are two here: one through `speaker`/`company`, one through `track`. Decompose each into its own table. For **each**, write out the concrete update, insertion, *and* deletion anomaly it caused before you removed it (three sentences per transitive dependency).

## Expected result

A set of small tables in which every non-key column depends on the key, the whole key, and nothing but the key. You should end with roughly these relations (names are yours):

- `speakers(speaker_id, name, email, company_id)`
- `companies(company_id, name, city)`
- `tracks(track, room)`
- `talks(talk_id, sub_id, title)` (or `submissions`/`talks` split)
- `submissions(sub_id, speaker_id, track)`

Confirm: pick any single fact ("Nimbus is in Austin") and verify it now lives in **exactly one row**.

## Done when…

- [ ] `normalize.md` has all four steps, each showing tables + FDs + the named anomaly.
- [ ] The 1NF fix eliminates the `;`-separated `talk_titles` cell entirely.
- [ ] The 2NF step explicitly identifies a *partial* dependency (or argues correctly why none exists after your 1NF split — depends on how you keyed it) .
- [ ] The 3NF step removes **both** transitive dependencies and writes all three anomaly types for each.
- [ ] You can point at any fact in the final schema and show it appears once.

## Stretch

- Does your final schema satisfy **BCNF**? Check every FD's left side — is it a superkey in its table? Write one sentence per table.
- Add a rule: "a company can now have offices in multiple cities, and a speaker works at a specific office." How does the model change? (Hint: `company_city` was a 1:1 fact; it just became 1:N.)

## Submission

Commit `normalize.md` to `c33-week-04/exercise-02/`.
