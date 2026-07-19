# Week 10 — Challenges

Two open-ended problems. Unlike the exercises, these have **no single right answer** — you make design calls and defend them. Do them after all three lectures and exercises.

1. **[Challenge 1 — Lock a multi-tenant schema](challenge-01-lock-a-multi-tenant-schema.md)** — take a shared, wide-open schema and make it *provably* tenant-isolated. Your deliverable includes an adversarial test suite that tries to break isolation and fails.
2. **[Challenge 2 — Audit a privilege model](challenge-02-audit-a-privilege-model.md)** — you inherit a database with a messy grant history. Find every over-grant, produce a written audit, and ship the least-privilege remediation.

## How these are judged

There's no autograder. Judge your own work (and swap with a peer if you can) against each challenge's **success criteria**, which reward:

- **Provable claims over hopeful ones.** "Tenants are isolated" is worth nothing without a test that *tries* to cross the boundary and is stopped.
- **Least privilege.** Every grant should be justifiable; "it works" is not the bar — "it works and nothing has more access than it needs" is.
- **Defense in depth.** The strong submissions layer roles + RLS + constraints so one hole is caught by the next layer.
- **A written rationale.** Security work you can't explain is security work no one can maintain.

Commit each challenge's SQL and write-up to your portfolio under `c33-week-10/`.
