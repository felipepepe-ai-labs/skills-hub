# Silent Failure Probe Report: <scope>

**Date:** <YYYY-MM-DD> · **Project:** <name> · **Commit:** <sha> · **Author:** <name>

## 1. Summary
- Points found: N (High X · Medium Y · Low Z).
- Probed: M. States: `ACTIVE` a · `LATENT` b · `COVERED` c.
- Handed to `root-cause-audit`: <ids or none>.

## 2. Scope and method
What was swept (modules, flows), stack, patterns checked and not applicable. Code mapped with the cbm graph. Data queries were read-only and run by the team.

## 3. Points by risk
| ID | File:line | Pattern | Impact | Reach | Likelihood | Score | State |
|---|---|---|---|---|---|---|---|

## 4. Evidence per probed point
### P<n> — <title>
- **Silent branch:** `file:line` and snippet.
- **Code probe:** test name, result (red/green).
- **Data probe:** query id, count today, count expiring soon.
- **State:** ACTIVE | LATENT | COVERED.
- **Recommended signal:** log with context | row in `trazas` | explicit error — and why.

## 5. Handoffs
| Point | To | What is passed (point, test, count) |
|---|---|---|

## 6. Not probed
Low-score points and why they were left out.

## 7. Recommendations
Ordered: first the `ACTIVE`, then `LATENT` by score. Owner and dependency for each.

## Annex A — Tests written (paths)
## Annex B — Queries and results
