# Audit Report: <case title>

**Date:** <YYYY-MM-DD> · **Project:** <name> · **Author:** <name> · **Status:** <open | closed | pending decisions>

## 1. Executive summary
- Symptom in one sentence.
- N causes found: X confirmed, Y discarded, Z inconclusive.
- Records affected: <count> of <total> (<%>).
- Decisions pending from the business: <count>.

## 2. Symptom
Expected behavior · Actual behavior · Environment · Since when · How it was detected.

## 3. Method
Source code mapped with the cbm graph (`<project>`, commit `<sha>`). Each hypothesis was accepted only with **data evidence** (query run by the team) and **code evidence** (`file:line`). Queries were read-only.

## 4. Inventory of involved code
| # | File:line | Role (read / write / guard / lookup) | Notes |
|---|---|---|---|

## 5. Causes
| ID | Cause | Pattern | Verdict | Data evidence | Code evidence | Affected |
|---|---|---|---|---|---|---|

### 5.x <ID> — <title>
- **What happens:** ...
- **Why it is silent:** ...
- **Proof:** query `T-H<n>`, result, and the code fragment that explains it.
- **Solutions:**

| Option | Description | Pros | Cons | Blast radius (callers/modules) |
|---|---|---|---|---|

- **Recommended:** <option> because <reason>.

## 6. Impact by cause
Table (and chart if useful): affected records per cause, overlaps, unattributed remainder.

## 7. Discarded and inconclusive
What was tried, why it was discarded, what is missing for the inconclusive ones.

## 8. Pending decisions
| # | Decision | Owner | Options | Blocks |
|---|---|---|---|---|

## 9. Remediation
Tool, modes (CHECK / SIMULATE / EXECUTE), results of the last runs (`logs` run id, counts), items intentionally not executed and why.

## 10. Next steps
Ordered list with owner and dependency.

## Annex A — Queries (`T-H<n>`) with results
## Annex B — Traceability: hypothesis → evidence → verdict
