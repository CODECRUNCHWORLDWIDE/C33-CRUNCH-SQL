# Mini-Project — 20 Business Questions from One Table

> Answer 20 real business questions about the Crunch company using **only** the `employees` table and the SQL you learned this week. One table, twenty questions, one clean deliverable — proof that you can query fluently.

**Estimated time:** 2.5–3 hours, best done Saturday after the exercises and challenges.

This is the week's capstone. A stakeholder doesn't hand you SQL — they hand you *questions*. Your value is turning each into a correct query, running it, and reporting the answer. You'll do that twenty times, which is exactly how many reps it takes for `SELECT` to stop feeling like translation and start feeling like thinking.

---

## Deliverable

A directory in your portfolio `c33-week-01/mini-project/` containing:

1. `answers.sql` — all 20 queries, each under a `-- Q<n>: <the question>` comment.
2. `report.md` — for each question: the answer (the actual number/rows), and one sentence of interpretation ("The top earner is Grace Hopper at $320,000").
3. `notes.md` — a short reflection (see the end).

Everything runs against the seed table from the [week README](../README.md). Works on PostgreSQL or SQLite; note which you used.

---

## The 20 questions

Group A and B are warm-ups; C and D need the full toolkit. Write one query per question.

### A. Counting and ranges

1. How many employees does Crunch have in total?
2. How many distinct departments are there?
3. How many employees are remote vs. office-based? *(Two numbers — you may use two queries this week.)*
4. What is the single highest salary, and who earns it?
5. Who are the five lowest-paid employees?

### B. Filtering

6. List everyone in Engineering, sorted by salary (highest first).
7. Who was hired in 2021?
8. Which employees are **not** based in the USA? Sort by country, then name.
9. List everyone earning between 90,000 and 130,000 inclusive.
10. Which employees have **no email address** on file?

### C. Pattern matching and text

11. List everyone whose last name starts with a letter from **A to H** (inclusive). *(Hint: `last_name < 'I'`, or a range — think about it.)*
12. Which employees have a `job_title` containing "Manager" or "Director"? Sort by title.
13. Show each employee's full name (`"First Last"`) and email, for those whose email is on file, ordered by last name.
14. Which employees live in a city whose name contains the letter "a" (case-insensitive)?

### D. Judgment, NULLs, and ordering

15. What is the average salary of the **Sales** department? *(One number.)*
16. List the Sales team with their commission rate, highest rate first, `NULL`s (if any) last.
17. Who are the three most recently hired employees, and when did they start?
18. Show the "second page" of employees when sorted by salary descending, 10 per page (i.e., ranks 11–20).
19. For every employee, show `first_name`, `salary`, and `annual_commission` = `salary * commission_pct`, treating a missing commission as **0** (not `NULL`).
20. Which employees are **remote AND earn above the company's typical pay** — where you define "typical" as "over 100,000" and state that assumption in `report.md`?

---

## Milestones

Pace yourself; don't try to do all 20 in one sitting.

- **Milestone 1 (45 min):** Questions 1–5. Get the environment smooth and the basic shape automatic.
- **Milestone 2 (45 min):** Questions 6–10. Filtering fluency.
- **Milestone 3 (45 min):** Questions 11–14. Pattern matching and text functions.
- **Milestone 4 (45 min):** Questions 15–20. The ones that require a decision or a `NULL`-safe touch — write your assumptions in `report.md` as you go.

---

## Rules

- **One table only.** No joins (that's next week). Everything is answerable from `employees` alone.
- **Name your columns and alias computed ones.** No naked `SELECT *` in the final answers.
- **Handle `NULL` deliberately** — questions 16 and 19 will punish you if you don't (`AVG` ignores `NULL`; `salary * NULL` is `NULL`).
- **Every question that involves a judgment call** (18's page size, 20's "typical") gets a one-line stated assumption in `report.md`.

---

## Rubric

| Criterion | Weight | "Great" looks like |
|-----------|------:|--------------------|
| Correctness | 40% | All 20 answers are right; counts verified |
| `NULL` handling | 20% | Q16/Q19 treat missing values correctly and deliberately |
| Readability | 15% | Aliases, parentheses, consistent formatting |
| Interpretation | 15% | `report.md` states the *answer in words*, not just the SQL |
| Stated assumptions | 10% | Judgment-call questions document the interpretation chosen |

---

## Reflection (`notes.md`, ~200 words)

1. Which question forced you to think hardest, and why?
2. Where did `NULL` almost bite you?
3. Which clause (`WHERE`, `ORDER BY`, `DISTINCT`, `LIMIT`) still feels least automatic? What will you drill this coming week?
4. One question you *couldn't* answer from a single table — what would you need? (Foreshadows Week 2 joins.)

---

## Why this matters

This mini-project is the shape of real analytics work: someone asks a question in English, and you return a trustworthy number. Do it twenty times and single-table querying becomes reflexive — which is exactly the foundation Week 2 (joins) builds on. Keep `answers.sql`; you'll extend these same questions across multiple tables next week.

When done: push, then take the [quiz](../quiz.md) and start [Week 2 — Joins & set operations](../../week-02-joins-and-set-operations/).
