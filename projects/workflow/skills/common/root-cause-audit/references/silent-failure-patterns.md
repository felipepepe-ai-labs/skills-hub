# Silent Failure Patterns

Catalog for phase 2 (search) and phase 3 (hypotheses). Each pattern lists what it looks like, how to find it with cbm/grep, and what data would prove it. A pattern that applies must produce a hypothesis; a pattern that does not apply must be stated as checked.

| # | Pattern | Looks like in code | How to find it | Data that proves it |
|---|---|---|---|---|
| 1 | **Missing config row** | Lookup in a general/parameter table whose absence skips the action | `search_code` on the config table or key; `trace_path` outbound from the writer | Expected keys vs. keys present: `LEFT JOIN` expected list to config, count the NULLs |
| 2 | **Incomplete mapping table** | A second lookup (A→B, e.g. zone→owner) returns nothing for some inputs | Readers of the mapping table inside the write path | Inputs of the real population with no row in the mapping |
| 3 | **State/validity filter leaves zero rows** | Query filters by role, status or end date (`active`, `titular`, `end_date IS NULL`) and nothing matches | `WHERE` clauses on status columns in the write path | Entities where all candidate rows are inactive; count per state |
| 4 | **Swallowed exception** | `catch {}`, `WHEN OTHERS THEN NULL`, `except: pass`, error logged at debug only | `search_code` for empty catch / `WHEN OTHERS` / `NULL;` | Matching error in DB/app logs; or none (that is the finding) |
| 5 | **NO_DATA_FOUND / empty result treated as OK** | `SELECT INTO` or `.first()` result is null and the code continues or returns | Reads followed by an early `return`/`EXIT` without log | Same as 1–3: which input produced the empty read |
| 6 | **Early return without log** | `IF cond THEN RETURN;` before the write, no trace | `get_code_snippet` of the writer; read guards from top | Cases where `cond` is true vs. records missing |
| 7 | **Wrong key/join** | Join on a column that is NULL or not unique for some rows | Join/ON clauses; mapping of FK columns | Rows lost by inner join: `COUNT(*)` with inner vs. left join |
| 8 | **Data-type/format mismatch** | Padding, trailing spaces, `41002.0` vs `41002`, case, leading zeros | Comparisons on codes (postal, ids) | Same value written two ways; `LENGTH`/`TRIM` differences |
| 9 | **Feature flag / environment toggle** | Behavior gated by a flag or per-env property | `search_code` for flag names and property readers | Flag value per environment at the time of the failure |
| 10 | **Race or ordering** | Write depends on another process having run first | `trace_path` callers and job schedulers | Timestamps: dependent write earlier than its prerequisite |
| 11 | **Dead path / never called** | Writer exists but no caller reaches it | `search_graph(max_degree=0, exclude_entry_points=true)` or `trace_path` inbound returns nothing | Absence of any record ever; no calls in logs |
| 12 | **Partial transaction / rollback** | Commit placed after a step that can fail | Transaction boundaries around the write | Parent row present, child row absent |

## How to use

1. For each pattern, run the "how to find it" step and note `applies / does not apply / not checkable`.
2. Applying patterns become hypotheses `H1..Hn` (phase 3) with `file:line`.
3. Real cases combine several: the same symptom often has **more than one** independent cause. Do not stop at the first confirmed one; a fix for cause A leaves the symptom alive if B exists.
4. Mark business-owned causes (mapping contents, which role is valid when one is inactive) as decisions, not bugs.
