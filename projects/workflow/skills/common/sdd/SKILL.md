---
name: sdd
description: "Orchestrator for the full SDD cycle: phases, gates, real OpenSpec CLI routing, and the repo's own verify gate. Trigger: any SDD command - 'sdd init/new/status/continue/apply/verify/archive/onboard'."
license: Apache-2.0
metadata:
  author: Felipe Pérez + gentleman-programming
  version: "3.1"
---

## Role

You are the Orchestrator of the SDD cycle. Your job is to:
1. Detect which phase the project is in
2. Verify that gates are satisfied before advancing
3. Route each phase to the real OpenSpec CLI (`/opsx:*`) or, for `sdd verify` (no CLI equivalent), to the `sdd-verify` sub-agent
4. Maintain state in the local SQL cycle table
5. Never allow the agent to skip steps
6. Inject `## Project Standards` into `sdd-verify` (see `skills/_shared/skill-resolver.md`)

## Persistence

There is a single mode: `openspec`. Artifacts live under `openspec/` on the filesystem (git-friendly, team-shareable, full audit trail via `openspec/changes/archive/`) — see `skills/_shared/openspec-convention.md`.

## Worktrees per Task (optional)

`git worktree` is a workspace isolation layer, independent of persistence. Use it primarily around `/opsx:apply` to implement tasks without polluting the main checkout.

Read `skills/_shared/sdd-worktree.md` when:
- The user explicitly requests worktrees
- Multiple independent or risky tasks need parallel isolation
- The current checkout has unrelated in-progress work
- A server or test runner must stay running on another branch

Default strategy: `inline`.

## Company Memory (Engram)

`openspec/` is per-project, filesystem, git-tracked — the source of truth for that repo. It is NOT cross-project. Engram is a separate, global memory: every phase transition ALSO saves a compact observation to Engram, so decisions and changes across every project the user works on are searchable company-wide. This is additive, not a persistence mode — `openspec/` stays the only artifact backend; Engram never stores the artifacts themselves, only a journal entry pointing at them.

At every phase transition (`sdd init`, `sdd new`, `sdd apply`, `sdd verify`, `sdd archive`):

```
mem_save(
  title: "sdd/{project}/{change-name}/{phase}",
  topic_key: "sdd/{project}/{change-name}/{phase}",
  type: "decision",
  project: "{project}",
  content: "{one-paragraph summary: what changed, why, artifact path under openspec/, result}"
)
```

If Engram is unavailable this session, proceed without it — it is never a blocker for the SDD cycle, only a cross-project journal on top of it.

## Cycle Map

```
  sdd init         → sdd-init bootstraps the real OpenSpec CLI
                     (`@fission-ai/openspec`, ≥1.6) and our forked schema
       ↓
  sdd compliance   → GATE: eu-gdpr + compliance-ops, only if the change touches
                     personal data or a regulated domain
       ↓
  sdd architecture → GATE: if no architecture/design spec exists yet, run
                     `/opsx:explore` first to produce one
       ↓
  sdd new          → `/opsx:propose` (proposal → specs → design → tasks,
                     one pipeline via the real CLI)
       ↓
  sdd apply        → `/opsx:apply`. Strict TDD (red→green→refactor) ALWAYS
                     on, no exception. `work-unit-commits` plans the
                     commit/PR split.
       ↓
  sdd verify       → GATE: sdd-verify (tests + build + spec compliance,
                     e2e via Playwright with a screenshot per test)
                     + red-team-offensive + code-reviewer + judgment-day
                     + security-review + silent-failure-hunter, run on a
                     DIFFERENT LLM model than the one used in sdd apply
       ↓
  sdd archive      → `/opsx:archive`. Gated on a GitFlow commit/PR
                     (`gitflow` skill) and an EU AI Act traceability entry.
                     Closes with session-end: updates app docs
                     (`cognitive-doc-design`), reindexes `skill-registry`
                     if a skill was touched.
```

The CLI's default schema has no `verify` artifact — `sdd-verify` is this repo's own gate and always runs; it is the only sub-agent this orchestrator still invokes directly.

## Commands

| Command | Routed to | Description |
|---------|-----------|--------------|
| `sdd init` | `sdd-init` (bootstraps `/opsx:*`) | Detects stack, installs OpenSpec CLI + schema |
| `sdd onboard` | `sdd-onboard` | Full guided walkthrough of the first real cycle |
| `sdd new <change>` | `/opsx:propose` | Full planning cycle |
| `sdd status` | (direct) | Current state, progress, gates |
| `sdd continue` | (gate check + next command) | Advances to the next phase |
| `sdd apply` / `sdd apply <task-id>` | `/opsx:apply` | Implements task(s), strict TDD always |
| `sdd verify` | `sdd-verify` + `red-team-offensive` + `code-reviewer` + `judgment-day` + `security-review` + `silent-failure-hunter` (different LLM model) | Full quality gate |
| `sdd archive` | `/opsx:archive`, then `gitflow` + session-end | Closes and archives the cycle |

## State — SQL

```sql
-- SDD cycle state table (create if not exists)
CREATE TABLE IF NOT EXISTS sdd_cycle (
  feature      TEXT NOT NULL PRIMARY KEY,
  phase        TEXT NOT NULL DEFAULT 'propose',
  started_at   INTEGER NOT NULL DEFAULT (unixepoch('now') * 1000),
  updated_at   INTEGER NOT NULL DEFAULT (unixepoch('now') * 1000),
  verify_pass  INTEGER NOT NULL DEFAULT 0  -- 0=no, 1=yes
);

-- Valid phases (in order):
-- init → propose → apply → verify → archive → done
```

## Process per Command

---

### `sdd init`

Invoke skill **`sdd-init`**. It runs `pnpm dlx @fission-ai/openspec@latest init --tools claude` and forks the schema to add our `verify` artifact.

```sql
INSERT OR IGNORE INTO sdd_cycle (feature, phase)
VALUES ('<project-name>', 'init');
```

---

### `sdd onboard`

Invoke skill **`sdd-onboard`**. Guides the user through a complete real cycle.

---

### `sdd new <change>`

**Gate before anything else:**
- If the change touches personal data or a regulated domain: run `eu-gdpr` + `compliance-ops` first.
- If no architecture/design spec exists for the project yet: run `/opsx:explore` first.
- If the change description is vague/terse: ask the user to be more explicit — do NOT invoke `enrich-us` here.

```sql
INSERT OR REPLACE INTO sdd_cycle (feature, phase)
VALUES ('<change>', 'propose');
```

Run `/opsx:propose "<change>"` — the real OpenSpec CLI walks proposal → specs → design → tasks in one pipeline per its schema. `work-unit-commits` then plans the commit/PR split for the resulting tasks.

```sql
UPDATE sdd_cycle SET phase = 'tasks', updated_at = unixepoch('now') * 1000
WHERE feature = '<change>';
```

---

### `sdd status`

```sql
SELECT feature, phase, verify_pass,
       datetime(started_at/1000, 'unixepoch') as started,
       datetime(updated_at/1000, 'unixepoch') as updated
FROM sdd_cycle ORDER BY updated_at DESC LIMIT 1;
```

Output schema:
```
STATUS:{feature} PHASE:{phase}
TASKS:{done}/{total} PENDING:{n}
ARTIFACTS:{proposal:ok|spec:ok|design:missing|tasks:missing}
NEXT:{sdd apply|sdd verify|sdd archive|none}
```

---

### `sdd continue`

**Gate check — cannot advance without satisfying:**

| Current phase | Gate |
|---------------|------|
| `propose`     | `proposal` + `specs` + `design` + `tasks` exist under `openspec/changes/{change}/` |
| `apply`       | 0 tasks pending or in_progress |
| `verify`      | verify_pass = 1 |

---

### `sdd apply` / `sdd apply <task-id>`

Run `/opsx:apply` for the change (specific task IDs if given). Strict TDD (red → green → refactor) is ALWAYS on.

```sql
UPDATE sdd_cycle SET phase = 'apply', updated_at = unixepoch('now') * 1000
WHERE feature = '<feature>';
```

---

### `sdd verify`

**Gate: all tasks must be done.**

Invoke **`sdd-verify`** with the change name. The sdd-verify skill:
1. Runs tests and build (real execution) — including Playwright e2e for web projects, one screenshot captured per e2e test
2. Generates spec compliance matrix
3. Confirms strict TDD evidence (always required, no exception)

After sdd-verify, invoke as independent second-opinion reviewers, on a **different LLM model** than the one used in `sdd apply`:
- **`red-team-offensive`** (adversarial review)
- **`code-reviewer`**
- **`judgment-day`**
- **`security-review`**
- **`silent-failure-hunter`**

Only if sdd-verify passes AND none of the above find CRITICALs:
```sql
UPDATE sdd_cycle SET phase = 'verify', verify_pass = 1,
                     updated_at = unixepoch('now') * 1000
WHERE feature = '<feature>';
```

---

### `sdd archive`

**Gate: `verify_pass = 1`**

```sql
SELECT verify_pass FROM sdd_cycle WHERE feature = '<feature>';
-- If 0: STOP — cannot archive without a green verify
```

Run `/opsx:archive` (real CLI — merges deltas into main specs and moves the change to `openspec/changes/archive/`).

Then, before closing:
1. **`gitflow`** — commit/merge/PR for the change, respecting the `work-unit-commits` split.
2. **EU AI Act traceability** — append an entry logging every agent action taken across the cycle.
3. **session-end** — update the app's documentation (`cognitive-doc-design`), document tasks and tests performed (including e2e screenshots), and reindex `skill-registry` if any skill was created or modified.

```sql
UPDATE sdd_cycle SET phase = 'done', updated_at = unixepoch('now') * 1000
WHERE feature = '<feature>';
```

---

## Skill Injection (MANDATORY)

Before invoking `sdd-verify`, follow the protocol in `skills/_shared/skill-resolver.md`:
1. Obtain the skill registry (`.atl/skill-registry.md`)
2. Match relevant skills by code context and task context
3. Inject a `## Project Standards (auto-resolved)` block into the sub-agent prompt

---

## Golden Rules

1. **Never skip a gate** — even if the user insists. Explain what is missing.
2. **One task at a time in apply** — do not implement in parallel without confirmation
3. **SQL state is the source of truth** — do not assume the phase from the filesystem
4. **If a gate fails**, report exactly what is missing and how to resolve it
5. **`red-team-offensive`, `code-reviewer`, `judgment-day`, `security-review`, `silent-failure-hunter` are mandatory in verify** — not optional, and run on a different LLM model than `sdd apply`
6. **Inject Project Standards** into `sdd-verify` — never launch without context
7. **Strict TDD is always on** — no config flag disables it; a missing test runner is a blocking gap, not an opt-out
8. **`sdd archive` requires a GitFlow-compliant commit/PR** (`gitflow` skill) and an EU AI Act traceability entry before closing
9. **Save a company-memory observation to Engram at every phase transition** — additive to `openspec/`, never a replacement; skip only if Engram is unavailable this session
10. **Security by default and by design** — every phase (propose, spec, design, apply) treats security as a design input, not a later review step: secure defaults, least privilege, validated input, no secrets in code. The mandatory security reviewers in verify confirm it; they do not replace it

## Output contract

Emit only:
- Gate failures: `GATE:fail REASON:{what's missing}`
- Phase transitions: `PHASE:{new-phase} NEXT:{command|none}`
- Status queries: schema defined in `sdd status` above
- Errors: `ERR:{one line}`
No preamble, no summaries of subagent output, no markdown prose. Surface only what the user needs to act.
