---
name: session-end
description: "Structured session close: separates Engram operational memory from Atlas knowledge, refreshes the codebase graph, compresses via Headroom, saves observations and optional journal entry. Trigger: 'close/end session', 'save to engram', 'save context', 'summarize what we did', or end of a long session."
license: Apache-2.0
metadata:
  author: Felipe Perez
  version: "1.2"
---

## When to Use This Skill

- The user explicitly asks to close/save at the end of a session
- Significant work was done (deploy, feature, bugfix, architectural decision)
- The user says "let's wrap up", "save what matters", "summarize the session"
- Decisions/learnings are detected that are not in persistent memory

Do NOT use if:
- The session was trivial (a typo, a question without code)
- session-end was already run in this same turn

---

## Closing Protocol

### Step 1 — Review the Session History

Go through the conversation and identify candidates. For each one, decide the type:

| Type | Save if... |
|------|-----------|
| `decision` | Non-obvious architecture/technology/approach decision |
| `bugfix` | Resolved error — symptom, root cause, fix |
| `architecture` | Designed/documented structure (module, network, system) |
| `pattern` | Repeatable convention or way of working |
| `config` | Environment configuration (nginx, systemd, DNS, env vars) |
| `discovery` | Undocumented finding (occupied port, limitation, unexpected behavior) |

**Do NOT save**:
- Obvious steps or officially documented behavior
- Code already in the repo (the repo is the source of truth)
- Temporary state (in-progress tasks, TODOs without context)
- Things already in Engram/Atlas — verify first

### Step 2 — Classify Destination: Engram vs Atlas

Each candidate goes to ONE destination (not both unless there is an explicit reason).

**Atlas — persistent and architectural knowledge** (useful life >6 months, cross-session):
- Lives in `/mnt/nas/Obsidian/` (Obsidian vault). Primary access via filesystem (Read/Write/Edit) — the `atlas_*` MCP may not be available in every session.
- Structure: `Proyectos/<project>.md` (entity page per project), `Stack/<category>/<tech>.md` (tech reference with `_INDEX.md` master), `Setup/` (infra/operational), `AI/`, `Temp/`.
- "Load-bearing" architecture decisions: stack choice, data model, stable API contracts.
- Organizational conventions that apply across multiple projects.
- Canonical mappings (e.g. "MDASH↔vi-sdd", OWASP taxonomies).
- Established personal/team working patterns.

**Most typical at session close**: if there was a spec/phase close with architectural substance, **update `Proyectos/<project>.md`** with the delta (current state, metrics, new Stack links). This is usually not a new page — it is an upsert on the existing entity page. Mandatory convention: refresh the `**Last updated:** YYYY-MM-DD` field at the top.

**Engram — operational memory** (useful life weeks-months, day-to-day):
- Resolved bugs (symptom+cause+fix)
- Spec state and metrics (temporal snapshots)
- Specific environment configuration (venv, ports, paths)
- Prioritized technical debt
- Tactical decisions within a spec

**Quick rule**: if the answer to "will this still be true in 6 months?" is YES → atlas. If "it depends on the current state" → engram.

### Step 2.5 — Codebase Graph Freshness (codebase-memory-mcp)

Only if the `codebase-memory` MCP tools are available this session AND source files (not just docs/memory) were created, edited, or committed:

- `list_projects` — check whether this repo has ever been indexed.
  - Never indexed → skip this step entirely. Do not suggest indexing a project that has never used the graph.
- If previously indexed: `index_status` — compare the indexed commit against the current HEAD (post-commit, if you committed this session).
  - Behind HEAD → ask the user: "Codebase graph is N commits behind — re-index with `index_repository`? (may take a while on large repos)"
  - Only run `index_repository` if the user confirms. Never re-index automatically — same caution as the Tests/Benchmark step in `session-start`.
  - Up to date → nothing to do, no need to mention it.

### Step 3 — Check for Duplicates Before Saving

For each candidate:

- Engram: `mem_search("<title or keywords>", project=<X>)`. If something similar exists: use `topic_key` to upsert with `mem_save` or `mem_update`.
- Atlas: check if `Proyectos/<project>.md` exists in `/mnt/nas/Obsidian/`. If yes → edit the relevant section (do not create a new page). If the observation spans several projects or concerns a technology, consider `Stack/<category>/<tech>.md`. Last resort: create a new page — only if the topic genuinely does not fit any existing one.

`topic_key` values (Engram) must be stable and compound: `vi-sdd/spec-006-owasp`, `homelab/nginx-config`, `personal/git-workflow`.

### Step 4 — Save with Canonical Structure

Mandatory format for `content` (engram and atlas):

```
**What**: [what was done or decided, in one line]
**Why**: [why it matters, what problem it solves]
**Where**: [paths, files, commands, affected services]
**Learned**: [gotchas, edge cases, decisions made — omit if not applicable]
```

**Titles**: short, searchable, prefixed by project when applicable.
- ✅ `vi-sdd: dedup command-injection-sink (spec 006 phase 8)`
- ✅ `homelab: nginx on pihole2 - reload without restart`
- ❌ `fixed a problem with nginx`

**Common projects**: `homelab`, `poc-trello`, `openclaw`, `sdd-office`, `vi-sdd`, `engram`.
**Personal scope** (preferred for infra and patterns that apply across projects): `scope: personal`.

Run saves in parallel when they are independent (multiple `mem_save` in a single turn).

### Step 5 — Project Journal (if applicable)

If the project has `docs/journal/sessions/` and the session was substantial (not trivial):

- Check if an entry for today already exists: `docs/journal/sessions/YYYY-MM-DD.md`
- If it does NOT exist and the session warrants an entry: **ask the user** before creating it
  - Template at `docs/journal/sessions/_template.md` if it exists
  - Minimum content: objective, work done, decisions, bugs found, quantified result, state at close
- If it already exists: ask whether to append the advances from this turn
- Do NOT create the journal automatically — always confirm

If the project does NOT have `docs/journal/` → skip this step.

### Step 6 — Summary to User

Display a table of what was saved:


| Destination | Title / Page | Type | Action |
|-------------|-------------|------|--------|
| Engram | ... | bugfix | save |
| Engram | ... | decision | update (#1234) |
| Atlas | `Proyectos/vi-sdd.md` | architecture | edit (section X) |
| Atlas | `Stack/Languages/Rust.md` | pattern | edit (section Y) |

**Security:** secure by default and by design applied to this session's implementation · pending findings: <list> | none

And briefly mention what was decided NOT to save and why (e.g. "trivial", "already in X").

If a journal was created/modified: include the file path.
If the codebase graph was re-indexed (Step 2.5): mention the new indexed commit; if it was flagged stale but the user declined to re-index, note that it's still pending.

---

## Operational Rules

- **Do not invent** observations — only save what actually happened in the conversation.
- **Do not duplicate** — always run `mem_search`/`atlas_search` first. Upsert with `topic_key`.
- **Parallelism**: independent saves go in the same turn.
- **Atlas-first**: when in doubt, default to Atlas. Only use Engram if the candidate is clearly temporal/tactical (a snapshot tied to current state, a resolved bug, a spec metric) per the Step 2 table.
- **Respect project language**: if the project's CLAUDE.md is in Spanish, titles and content go in Spanish too.
- **Do not silence errors**: if engram/atlas fails, report to the user what was left pending.
- **Security by default and by design**: at close, check that the implementation done this session took security into account from the start (secure defaults, least privilege, validated input, no secrets in code). Save security decisions and findings as normal candidates (Step 1), report any pending finding in the summary, and never persist secrets, keys or tokens in Engram, Atlas or the logbook.

---

## Quick Heuristics

**Is this worth saving?**
- Would you search for this in memory next time you do the same thing? → Yes → save
- Did it take >15 min to resolve or understand? → Yes → save
- Is it something you could forget in 2 weeks? → Yes → save
- Is it already in the code, CLAUDE.md, or docs/? → No → don't save

**Engram or Atlas?**
- "The vi-sdd OWASP catalog maps 19 bug_classes to 7 categories" → atlas (stable taxonomy)
- "Spec 006 closed on May 21 with F1 0.998 and 317 tests" → engram (temporal snapshot)
- "To avoid duplicates, the command-injection-sink dedup omits the symbol" → engram (tactical decision)
- "We use a 6-stage pipeline architecture inspired by MDASH" → atlas (architectural decision)

**Create a journal?**
- Session <30min with no substantial changes → no
- Session with new bug fix, decision, or closed phase → yes (ask first)
- Conversation/analysis only without code → no, but update memory

---

## Compatibility

This skill **replaces** the old `engram-fin-sesion`. If the user invokes that by name, this skill covers the same trigger and adds Atlas + journal.
