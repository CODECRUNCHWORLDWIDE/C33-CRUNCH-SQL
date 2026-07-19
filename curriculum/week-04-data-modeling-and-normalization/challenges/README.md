# Week 4 — Challenges

Two open-ended problems. Unlike the exercises, these have **no single right answer** — a good solution is one you can *defend*. The gap between "can normalize a given table" and "can design a schema from scratch" lives here.

1. **[Challenge 1 — Design a schema from a spec](challenge-01-design-from-a-spec.md)** — read a real-world business description and produce a complete normalized schema, keys and constraints included.
2. **[Challenge 2 — Spot and fix normalization violations](challenge-02-spot-and-fix-violations.md)** — audit a badly-designed schema, name every violation, and repair it — with a written argument for each change.

Do them after the three exercises. Bring your ER and normalization skills together; there's no scaffolding this time.

## How these are judged

There's no autograder. Judge your own work (or trade with a peer) against:

- **Correctness** — does the model actually capture the domain? Can you store every fact the spec mentions, and *only* legal facts?
- **Normal form** — is it at least 3NF, with every deliberate exception justified in writing?
- **Constraints** — are the business rules enforced by the *schema*, not left to hope?
- **Defensibility** — for every non-obvious decision (surrogate vs. natural key, a denormalization, an `ON DELETE` choice), can you state the trade-off out loud?

A schema that's "correct but undefended" scores lower than one with a clear rationale. Write your reasoning down.
