# Engram Protocol

Save at the end of every phase so the audit can be resumed in another session and reused on similar cases.

## Rules

- Before saving: `mem_search("<keywords>", project=<project>)`. If the case exists, upsert with the same `topic_key`.
- Always pass `project` explicitly (the gateway falls back to a default project otherwise).
- `topic_key` format: `rca/<project>/<case-slug>/<phase>` with phase in `intake | inventory | hypotheses | verdicts | report | remediation`.
- Body in the canonical format:

```
**What**: <what this phase established, one line>
**Why**: <why it matters for the case>
**Where**: <files, tables, queries, doc paths>
**Learned**: <gotchas, patterns, decisions; omit if none>
```

- Use `capture_prompt: false` for automated phase records.
- Save conclusions and evidence summaries, not raw dumps. Do not save secrets, credentials or customer data; use counts and identifiers of code, not personal data.
- If a memory call fails, report what was left unsaved and continue.

## What to save per phase

| Phase | Save |
|---|---|
| intake | Symptom, expected vs. actual, environment, prior similar cases found |
| inventory | Entry points and write/guard/lookup sites with `file:line` |
| hypotheses | `H1..Hn` with pattern, confidence, supporting `file:line` |
| verdicts | Per hypothesis: verdict, counts, data evidence, code evidence |
| report | Confirmed causes, solutions chosen/proposed, `PENDING DECISION` items and owner |
| remediation | Tool name, modes, run IDs from `logs`, counts per cause, items not executed |

## Closing

Call `mem_session_summary` once, after the report. Recurring patterns discovered (a new silent-failure pattern, a project-specific gotcha) go as their own observation with `topic_key = rca/<project>/patterns` so the next audit starts with them.
