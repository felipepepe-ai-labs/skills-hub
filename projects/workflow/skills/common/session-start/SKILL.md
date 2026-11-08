---
name: session-start
description: "Session resumption briefing: detects project, git state, last journal, active spec, Engram/Atlas context, and codebase graph. Trigger: 'start session', 'resuming', 'where did we leave off', 'project state', 'bring me up to speed'."
license: Apache-2.0
metadata:
  author: Felipe Perez
  version: "1.8"
---

## When to Use This Skill

- The user opens a new session on a project and wants to resume context
- The user says "where did we leave off", "bring me up to speed", "project state"
- After several days away from a project, when returning to it
- Before starting to code in a non-trivial repo

Do NOT use if:
- The user asks something specific that does NOT require reloading project context
- session-start has already been run in this same session

---

## Startup Protocol

Execute steps in this order. Codebase Graph (Step 2) runs right after project detection, genuinely reading the project's structure — not gated behind git/docs/memory. **Parallelize** independent calls in the same turn where order doesn't matter (git + reads + mem_context + atlas_search are all independent of each other).

### Step 1 — Detect the Active Project

- `pwd` and look for project markers: `.git`, `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `composer.json`
- If there is a `CLAUDE.md` in the directory: read it (policies, conventions, venv, commands)
- If there is a `CLAUDE.md` in `~/.claude/`: already loaded automatically — do not re-read
- Canonical project name = directory name or `package.json::name`/`pyproject.toml::project.name`

### Step 2 — Codebase Graph (codebase-memory-mcp), read FIRST

Do this right after project detection, unconditionally when the `codebase-memory` MCP is available — do not gate it behind "the user asked something structural."

- `list_projects` — check whether the current repo is indexed.
- If NOT indexed → note "Codebase graph not indexed for this project" and continue with the rest of the protocol (do not index here — that's `session-end`/explicit-request work).
- If indexed → call `get_architecture` (project structure, layers, entry points) and, if useful for this project's shape, `search_graph`/`trace_path` for the hotspots. Use this to genuinely understand the codebase's real structure before reading docs or Engram — not just to check staleness.
- Separately, call `index_status` and compare the indexed commit against the HEAD commit from Step 3 → if behind, flag "Codebase graph stale, N commits behind" in the briefing. Staleness does not block using `get_architecture` — a few commits behind is still far more useful than skipping it.

CAUTION: if the `codebase-memory` MCP is not connected this session, skip this step silently — do not block the briefing on it.

### Step 3 — Git State

Run in parallel in a single turn:

- `git status --short` (uncommitted changes)
- `git branch --show-current` + `git log --oneline -5` (branch + last 5 commits)
- `git stash list` (pending stashes)

If there are uncommitted changes or stashes → flag them in the briefing.

### Step 4 — State Documentation

Read if they exist (in order, without failing if any is missing):

1. `docs/journal/sessions/` — most recent entry by date
2. `docs/STATE.md` — executive snapshot
3. `IMPLEMENTATION_SUMMARY.md` — capabilities and spec state
4. `README.md` — only if the above do not exist

### Step 5 — Active Spec (if the project follows SDD)

- Look for `specs/NNN-*/tasks.md` (descending numeric order — the last is usually the active one)
- For each spec, count `- [ ]` checkboxes (pending) vs `- [x]` (closed)
- Active spec = the last one whose pending ratio > 0 AND whose last edit is recent

CAUTION: unchecked checkboxes may be stale. Cross-reference with `IMPLEMENTATION_SUMMARY.md` or the journal before stating "spec X has Y pending tasks". If the summary says "closed" and checkboxes are unchecked → report the conflict, do not assume it is open.

### Step 6 — Persistent Memory (Atlas-first)

Fetch in parallel, but treat **Atlas as the primary source of truth** — it's where `session-end` now saves by default:

- Direct read of the **Atlas (Obsidian)** vault at `/mnt/nas/Obsidian/`:
  - `Proyectos/<name>.md` — project entity page, if it exists (architectural state).
  - If the `atlas_search` MCP is available, also use it. If not, use `grep`/`find` in the vault.
- `mem_context(project=<name>, limit=20)` — recent observations in Engram, for temporal/tactical detail Atlas doesn't carry (bugfixes, spec snapshots, in-progress state).

If Atlas and Engram disagree on the same fact, **Atlas wins** — it's the later, more deliberate write. Use Engram to fill gaps (what happened recently) rather than to override Atlas's architectural state.

Cross-check dates:
- If `Proyectos/<name>.md` has `Last updated:` earlier than the project's last journal → flag Atlas drift in the briefing (reconciliation is `session-end` work).
- If the last Engram observation is older than the last journal → flag "Engram outdated, N days of drift" (lower priority than Atlas drift).

### Step 7 — Tests/Benchmark (optional, ask first)

Do NOT run automatically. Ask the user:

> "Shall I run tests + benchmark as a sanity check? (may take ~30s-2min)"

If they say yes:
- Detect the test command from the project's `CLAUDE.md` or conventions (`pytest`, `pnpm test`, `go test ./...`, `cargo test`, `unittest discover -s tests`)
- Respect venvs and paths documented in `CLAUDE.md` (e.g. vi-sdd uses `/home/sandman/.venvs/vi-sdd/bin/python`)
- Report green/red + number of tests; if red, do NOT attempt to fix — only flag it

### Step 8 — Briefing to User

Return a structured summary:

```
## Resumption point — <project>

**Branch:** <branch> · **Last commit:** <hash> <msg>
**Uncommitted changes:** <N files> | <none>
**Last session:** <journal date> — <title or first line>
**Codebase graph:** indexed as of <commit/date> | stale (N behind) | not indexed | unavailable
**Security:** secure by default and by design apply to all implementation · pending findings: <list> | none

### Project state
- <bullet with key metrics from STATE/IMPLEMENTATION_SUMMARY>
- <bullet with active spec if any>
- <bullet with architecture/entry-points summary from `get_architecture` when the codebase graph is indexed — layers, entry points, where symbols concentrate>

### Prioritized debt (top 3)
- <from STATE.md §debt or IMPLEMENTATION_SUMMARY>

### Suggested next step
<1-2 concrete sentences based on the last journal and pending tasks>
```

**Onboarding variant — first contact.** When the first-contact heuristic fires (see
Heuristics), the resumption template above makes no sense ("last session: none").
Replace it with an onboarding briefing that teaches the project instead:

```
## First contact — <project>

### What it is
<README/CLAUDE.md: purpose in 2-3 lines>

### Stack & conventions
<manifests: language, framework, package manager, test runner>
<CLAUDE.md rules that constrain how work is done here>

### Structure
<directory tree / README description: layers and where symbols concentrate>

### Entry points & hotspots
<main/server/app + top fan-in symbols — where to start reading>

### How to run / test
<build, test, dev commands — from CLAUDE.md or package scripts>

### Health signals
<tests present?, CI?, uncommitted work?>
```

If conflicts are detected (outdated engram, checkboxes vs IMPLEMENTATION_SUMMARY, red tests), add an **⚠ Attention** section with the detail.

After the briefing, always end with a **session intake** — ask these questions to collect everything needed before starting work. Present them as a compact block, not a wall of text:

```
### Before we start — a few questions:

1. **Goal** — What do you want to accomplish today?
2. **Constraints** — Any deadline, scope limit, or thing to avoid?
3. **Approach** — Should I propose a plan first, or dive straight in?
   - Need SDD cycle? (spec, tasks, phases)
   - Architecture decision that warrants DDIA tradeoff analysis?
   - Something else I should load before starting?
4. **Blockers** — Anything waiting on a PR, external dependency, or another person?
```

**Onboarding intake**: on first contact, prepend one extra question — "Is this your
first time in this repo, or are you resuming work started elsewhere (no memory
recorded on this machine)?" Engram-first-contact is not always user-first-contact;
if they are resuming, keep the onboarding structure but skip the teaching tone.

STOP after showing the intake. Do not assume answers, propose code, or begin any task until the user replies. Questions 2–4 are optional — if the user only answers question 1, that is enough to proceed.

---

## Operational Rules

- **Write nothing to disk** during session-start. Read only.
- **Do not modify memories** (engram/atlas) in start — that is `session-end` work.
- **Parallelize reads** whenever possible. Bash + Read + mem_context + atlas_search can go in the same turn.
- **Fail gracefully**: if engram, atlas, codebase-memory-mcp, or any file does not respond/exist → omit that section from the briefing, do not break the flow.
- **Respect project language**: if CLAUDE.md specifies Spanish (like vi-sdd, homelab), the briefing goes in Spanish. English by default.
- **Do not suggest code changes** in the briefing. The next-step suggestion is "which task to tackle", not "which line to edit".
- **Security by default and by design**: every implementation that follows this session must take security into account from the start (secure defaults, least privilege, validated input, no secrets in code), not as a later review step. Surface any pending security finding found in STATE/journal/IMPLEMENTATION_SUMMARY in the briefing.

---

## Heuristics

**Is this first contact? (onboarding mode)**
- ALL of these → first contact: mem_context has no observations for the project,
  no journal/STATE/IMPLEMENTATION_SUMMARY, no Atlas entity page. Use the onboarding
  briefing variant (Step 8) and prepend the first-time intake question.
- Only SOME signals missing (e.g. journal exists but Engram is empty) → resumption
  briefing; flag the missing store as drift instead of switching template.

**Is there an active spec?**
- `specs/NNN-*/tasks.md` exists with `- [ ]` pending AND the journal/STATE does not explicitly say "closed" → yes
- If `IMPLEMENTATION_SUMMARY.md` says "closed on <date>" and checkboxes are unchecked → stale checkboxes, NO active spec

**Is Engram outdated?**
- Last observation with `project=<X>` > 7 days before the last journal → outdated
- Suggest at the end of the briefing: "Consider running `/session-end` at the end of today to sync Engram"

**Is Atlas relevant?**
- The Atlas vault lives in `/mnt/nas/Obsidian/`. Structure: `Proyectos/<project>.md` (entity page), `Stack/<category>/<tech>.md` (tech catalog with `_INDEX.md`), `Setup/` (infra), `AI/`, `Temp/`.
- If `Proyectos/<project>.md` exists → always read it: it has stable architectural state + backlinks to relevant Stack technologies.
- For small personal projects without an entity page, skip.

**Is the codebase graph stale?**
- `index_status` commit older than the HEAD commit from Step 3 (Git State) → stale, report the gap in commits.
- Not indexed at all is normal for a first session on a project — do not treat it as an error, just report it.
- Never call `index_repository` during session-start — indexing is a write/compute action reserved for `session-end` (with confirmation) or an explicit user request.

---

## Example Output

See `references/example-output.md` for a fully worked example (vi-sdd, simulated) — the format itself is already specified by the templates in Step 8.
