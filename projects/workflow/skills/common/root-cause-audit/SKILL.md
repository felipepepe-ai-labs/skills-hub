---
name: root-cause-audit
description: "Evidence-based root-cause audit of an error or silent failure: finds every candidate cause in the source code (cbm graph), validates each with read-only queries the user runs, and ends in a report plus Engram records. Trigger: 'root cause', 'why does it fail', 'por qué falla', 'auditar error', 'fallo silencioso', 'no guarda', 'investigar issue', 'find all causes of this error'."
license: Apache-2.0
metadata:
  author: Felipe Perez
  version: "1.0"
---

## When to Use

- A symptom has no visible error: something is not saved, not sent, not created, or comes out empty.
- An error has several plausible causes and none may be assumed true without proof.
- The result must be an auditable report, not just a fix.

## Scope Guard

- Not for a bug whose cause is already known and fixed in one file: just fix it.
- This skill never runs DML/DDL and never executes remediation on its own. The user runs every query and pastes the result back.
- Source of truth for code is the repo plus the cbm graph; for data, the results the user pastes. Never invent either.

## Core Rules

1. **No hypothesis becomes a cause without double evidence**: a code fragment (`file:line`) AND a data/log result. One without the other = `INCONCLUSIVE`.
2. **Phase 3 does not propose solutions.** Solutions come only after validation (phase 5).
3. **All generated queries are read-only** (`SELECT`, log greps, HTTP GET, running existing tests). See `references/validation-protocol.md`.
4. **Human gate** after phases 3, 4 and 6. Stop and wait; one question at a time.
5. Save to Engram at the end of every phase (`references/engram-protocol.md`). If Engram fails, say so and continue.
6. Answer in the user's language; keep code identifiers untouched.

## Workflow

### Phase 0 — Intake
Ask only what is missing: symptom, expected vs. actual behavior, where it happens (env, entity, table, endpoint), since when, what was already tried. Then `mem_search` for similar past cases (topic `rca/<project>/...`) and reuse prior findings.

### Phase 1 — Index (cbm)
1. `list_projects`. If the repo is **not indexed**, run `index_repository`, wait until done (`index_status`) and **continue with the audit**. Stop only if indexing fails; report the error.
2. If indexed but stale vs. HEAD, re-index the same way before mapping.
3. `get_architecture` once to orient yourself.

### Phase 2 — Map
Find every place involved in the symptom, not only the obvious one:
- `search_graph(name_pattern/label)` for entry points, handlers, jobs, repositories, procedures.
- `search_code` for literals: table/column names, error messages, config keys, flags.
- `trace_path(direction="both")` for callers/callees; `get_code_snippet` for exact source.
- Output an **inventory**: `file:line | what it does | read / write / guard / config lookup`. Compress big results (`headroom` skill) if long.

### Phase 3 — Hypotheses (gate)
For each candidate cause `H1..Hn` give: ID, one-line statement, `file:line` + snippet that supports it, matched pattern (`references/silent-failure-patterns.md`), confidence (high/medium/low), and what data would confirm or refute it. Cover every pattern in the catalog that applies. Mark gaps you could not cover. **No fixes here.** Ask the user to confirm/trim the list.

### Phase 4 — Validate (gate, loop)
Per hypothesis, produce the test from `references/validation-protocol.md`: read-only query, expected result if true, expected result if false. The user runs it and pastes the output. Then cross-check with code (cbm/Read): does the code path explain exactly what the data shows?

| Data | Code explains it | Verdict |
|---|---|---|
| matches | yes | `CONFIRMED` |
| contradicts | — | `DISCARDED` |
| matches | no | `INCONCLUSIVE` (a cause you have not seen yet: go back to phase 2) |

Record verdict, counts and evidence per hypothesis. Repeat until none is pending.

### Phase 5 — Audit document
For each confirmed cause: evidence (data + code), affected count, solutions with tradeoffs, and **blast radius** (`trace_path(direction="inbound")` on the code to change). Separate decisions that belong to the business from technical ones and list them as `PENDING DECISION`. Use `assets/audit-report-template.md`.

### Phase 6 — Remediation tool (optional, gate)
Only if the user wants to repair the affected records. Build it from `assets/remediation-tool-template.md`: modes `CHECK` → `SIMULATE` → `EXECUTE`, summary table `logs`, per-item table `trazas`. Items awaiting business decision are listed as `PENDING DECISION` and **never executed**. `EXECUTE` must refuse to run without a prior `SIMULATE` recorded in `logs`. After the fix, re-run the phase 2 sweep for the same pattern to catch siblings. To sweep the rest of the code proactively for the same shapes, with code tests and data checks, offer the `silent-failure-probe` skill.

### Phase 7 — Report and close
Produce the final report (`.docx` via the `docx` skill if available, otherwise Markdown) with numbers, charts where they help, confirmed/discarded table and pending decisions. Then `mem_session_summary`.

## Output Contract

Per phase, short and structured: tables for inventory, hypotheses and verdicts. Always end a gated phase with the single question the user must answer.
