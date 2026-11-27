# Risk Scoring

Rank candidate points so the user probes the most dangerous first. Three factors, 1 to 3 each. Score = impact × reach × likelihood (1 to 27).

| Factor | 1 | 2 | 3 |
|---|---|---|---|
| **Impact** (what is lost if the branch fires silently) | Cosmetic or recoverable log/metric | Business record missing but reconstructible | Data loss, wrong money/legal result, or record that cannot be rebuilt |
| **Reach** (how many flows pass through the point) | One caller | 2–5 callers | 6+ callers or a shared write path |
| **Likelihood** (chance the triggering data is absent) | Input is system-generated and always present | Maintained by people or a batch | Maintained manually, no owner, or known to drift (config, mappings, roles with end dates) |

## Getting reach

`trace_path(function_name, direction="inbound", depth=3)` on the function that contains the silent branch. Count distinct entry points (handlers, jobs, procedures), not call edges. Cross-service callers need `direction="both"`.

## Getting likelihood

- Lookups into config or mapping tables maintained by hand → 3.
- Filters on status, role or validity dates → 2 or 3 depending on how often records expire.
- Values produced by the same transaction → 1.
- If a previous audit (`rca/<project>/patterns`) saw this table drift, add one level.

## Bands

| Score | Band | Action |
|---|---|---|
| 18–27 | High | Probe first; code test and data query both |
| 8–17 | Medium | Probe if time allows; data query at least |
| 1–7 | Low | List in the report, do not probe |

State the three factors next to the score so the user can challenge them. Scores are a ranking aid, not a measurement.
