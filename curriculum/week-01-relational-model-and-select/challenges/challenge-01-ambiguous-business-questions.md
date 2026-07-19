# Challenge 1 — Ambiguous Business Questions

**Time:** ~60 minutes. **Difficulty:** Medium. **No single right answer.**

## The scenario

You're the data person at Crunch. Your VP drops by your desk and asks questions the way real people ask them — vaguely, in English, assuming you'll "just know" what they mean. Your job is the hardest and most valuable skill in this entire course: **turn a fuzzy question into a precise, defensible query.**

Half the work is *deciding what the question means*. There is no answer key. A senior engineer and a junior engineer will write different `WHERE` clauses for the same sentence — and the senior one **writes down the assumption they made** so the VP can correct it.

## Your task

For each question below:

1. **State your interpretation** in one sentence. Where's the ambiguity? What did you decide it means?
2. **Write the query** against `employees`.
3. **Note one alternative reading** you rejected, and why.

Put all of this in `challenge-01.md`.

## The questions

1. **"Who are our senior people?"**
   *(Ambiguity: senior by title? by salary? by tenure? by age? Pick one — or argue for a combination — and defend it.)*

2. **"Show me the well-paid remote folks."**
   *(What's "well-paid" — an absolute threshold? Above some rank? And is a `NULL` in a relevant column in or out?)*

3. **"I need the new hires."**
   *("New" since when? Last 12 months from today? Since Jan 1? Their first year? Today's date is fixed when you run it — say what "now" you used.)*

4. **"Which sales reps are underperforming on commission?"**
   *(Careful: only Sales has a `commission_pct` at all. What does "underperforming" even mean with the columns you have — lowest rate? Below the team's typical rate? Be honest about what this one table can and cannot answer.)*

5. **"Give me everyone in the Americas."**
   *(Which countries in this dataset count as "the Americas"? List them explicitly in your `IN (...)`. What did you include, what did you exclude, and could a reasonable person disagree?)*

6. **"Pull the people without a company email."**
   *(This one has a `NULL` trap baked in. What's the correct predicate, and what would the *wrong* one — `email = ''` or `email = NULL` — do instead?)*

## Constraints

- One table only (`employees`). If a question genuinely **can't** be answered from this table, say so explicitly — that's a valid and valuable answer.
- Every ambiguous question must ship with your written interpretation. A query with no stated assumption is an incomplete answer, even if it runs.
- Prefer readable SQL: parentheses around mixed `AND`/`OR`, aliases on computed columns.

## Hints

<details>
<summary>On "senior" (Q1)</summary>

There are at least four defensible signals in this table: `job_title` (contains "Senior", "Staff", "VP", "Director", "Manager", "C*O"?), `salary` (top quartile?), `hire_date` (tenure), and `birth_date` (age — legally fraught to use for staffing, worth a sentence on *why you'd avoid it*). A strong answer picks one, justifies it, and acknowledges the others.

</details>

<details>
<summary>On "the Americas" (Q5)</summary>

In this dataset the countries are USA, Canada, UK, Germany, Spain, Brazil, and India. "The Americas" clearly includes USA, Canada, Brazil. The interesting part is being explicit and consistent — hard-code the list, don't hand-wave.

</details>

## How success is judged

| Signal | Weak answer | Strong answer |
|--------|-------------|---------------|
| Interpretation | Jumps straight to SQL | States the chosen meaning in one clear sentence |
| Precision | Vague or overly broad `WHERE` | Predicate matches the stated interpretation exactly |
| `NULL` awareness | Ignores missing values | Handles or explicitly excludes `NULL`s where they'd distort the answer |
| Humility | Pretends the table can answer anything | Flags questions the single table genuinely can't answer (Q4) |
| Alternatives | One reading, no reflection | Names a rejected reading and why |

The best submissions read like a short conversation with the VP: "Here's what I think you meant, here's the answer under that reading, and here's what I'd need if you meant something else."

## Submission

Commit `challenge-01.md` to your portfolio under `c33-week-01/challenge-01/`.
