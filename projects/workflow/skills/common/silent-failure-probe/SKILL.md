---
name: silent-failure-probe
description: "Proactive scan for places where code can fail silently (missing config or mapping rows, empty lookups, inactive roles, swallowed errors), ranked by risk, with code tests and read-only data queries to prove them before they happen in production. Companion of root-cause-audit. Trigger: 'pruebas proactivas', 'dónde puede fallar en silencio', 'silent failure scan', 'prevenir fallos silenciosos', 'auditoría preventiva', 'before it breaks'."
license: Apache-2.0
metadata:
  author: Felipe Perez
  version: "1.0"
---

## When to Use

- No symptom yet: find where a silent failure could happen and prove it with tests and data checks.
- After a `root-cause-audit` closed: sweep the rest of the code for the same shapes.
- Before a release, migration or data load that touches write paths.

## Scope Guard

- With a live symptom, use `root-cause-audit` instead. This skill hands over to it when a point is already broken (`ACTIVE`).
- Never modifies production code. It writes only new test files, and only after approval.
- Data queries are read-only and the user runs them. Never invent counts.
- Zero findings is a valid result. Do not manufacture risks.

## Core Rules

1. A point becomes `ACTIVE` only with **double evidence**: silent code (`file:line`, failing test) AND a data count > 0.
2. A failing probe test is the finding, not a mistake: it proves the code stays silent.
3. Human gate after phases 3 and 5. One question at a time.
4. Save to Engram at the end of each phase (`topic_key rca/<project>/probe/<phase>`). If Engram fails, say so and continue.
5. Answer in the user's language; keep identifiers untouched.

## Workflow

### Phase 0 — Scope
Ask only what is missing: repo, module or business flow to sweep, and the stack (language, DB, test framework). `mem_search` topic `rca/<project>/patterns` and reuse known patterns.

### Phase 1 — Index (cbm)
`list_projects`. If the repo is **not indexed**, run `index_repository`, wait (`index_status`) and **continue**. Stop only if indexing fails. Re-index first if stale vs. HEAD.

### Phase 2 — Sweep
For every pattern in the catalog `../root-cause-audit/references/silent-failure-patterns.md`, run its search signal: `search_code`, `search_graph`, `trace_path`, `get_code_snippet`.
- Swallowed errors (catch/`WHEN OTHERS`/async): delegate to the `silent-failure-hunter` agent if installed; otherwise apply its targets inline (`references/code-probes.md`, last section).
- Output points `P1..Pn`: `file:line | pattern | snippet | what is lost if it fires`. Mark patterns checked and not applicable.

### Phase 3 — Risk (gate)
Score each point (`references/risk-scoring.md`): impact × reach × likelihood. Present them ranked and ask which to probe.

### Phase 4 — Code probes
For each selected point write a test that forces the silent branch and **asserts that a warning, error or trace is produced** (`references/code-probes.md`). Detect the project's test framework. Show the tests, write them only after approval, run them. Red = code is silent.

### Phase 5 — Data probes (gate)
For each point give a read-only query that counts **today** the entities that would hit that branch (`references/data-probes.md`, rules from `../root-cause-audit/references/validation-protocol.md`). The user runs it and pastes the result.

### Phase 6 — Classify
| Code silent | Data count | State |
|---|---|---|
| yes | 0 | `LATENT` — fix the warning before data arrives |
| yes | > 0 | `ACTIVE` — hand over to `root-cause-audit` with point, test and count |
| no (test green) | any | `COVERED` |

### Phase 7 — Report and close
Fill `assets/probe-report-template.md`: ranked table, tests, queries and counts, handoffs, recommended warning per point (log with context, `trazas` row, or explicit error). Save new patterns to `rca/<project>/patterns`. Then `mem_session_summary`.

## Output Contract

Tables per phase (points, scores, verdicts). End each gated phase with the single question the user must answer.
