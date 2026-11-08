---
name: atlas-docs
description: >
  Creates and updates documents in the Atlas (Obsidian) vault on the .casa intranet.
  Ensures all documents have bidirectional wikilinks for the knowledge graph.
  Trigger: when the user asks to create or update documents in atlas.casa, the
  Obsidian vault, or when something new is implemented on the intranet that needs
  to be documented.
license: Apache-2.0
metadata:
  author: Felipe Pérez
  version: "1.1"
---
> **🏠 CASA LAYER** — Only use this skill when working in a `.casa` intranet project.
> **Detect context by**: presence of `.casa` file in project root, `*.casa` domain references (maya.casa, infisical.casa, pihole.casa), or user explicitly mentions "intranet" or "home project".
> **Work/external projects**: do NOT suggest or use this skill. Skip silently.

## When to Use

- The user says "update the atlas.casa docs" or "document this in Atlas"
- Something new was just implemented on the intranet (new service, domain, feature)
- The user asks to create a new note in the vault
- A note is detected without links to other related notes
- The user asks about documentation state

## Vault Infrastructure

| Data | Value |
|------|-------|
| **Machine** | maya — `192.168.1.55` (the vault lives on the NAS, mounted locally) |
| **Vault path** | `/mnt/nas/Obsidian/` — read and write it **directly with the file tools; no SSH, no `sudo`** |
| **Owner** | `felipe` (the same user the agent runs as) |
| **App** | `atlas.casa` → `obsidian-api` on maya (`https://maya:20000`, code in `/mnt/nas/sources/obsidian-api/`) |

> **Do not use `pihole2` (`192.168.1.54`) or `/home/sandman/Obsidian/` for the vault.** That was the old location. pihole2 is only the nginx / DNS host for `.casa` domains; writing there would touch a machine that is not the vault and needs separate authorization (remote operation).

### Folder Structure

Verified against the real vault (see `Setup/Atlas-Vault-Structure.md`). The folder is `Proyectos/`, **never** `Projects/` (it does not exist).

```
/mnt/nas/Obsidian/
├── Stack/
│   ├── _INDEX.md          ← Master index of the entire vault
│   ├── AI-ML/             ← LangChain, Ollama, CrewAI, RAG, Whisper, Vosk...
│   ├── Auth/              ← JWT, BcryptJS, TOTP-2FA
│   ├── Backend/           ← Express, FastAPI, SpringBoot, Hono, Kafka
│   ├── Databases/         ← PostgreSQL, SQLite, MySQL, ChromaDB
│   ├── DevOps/            ← Docker, GitHubActions, Prometheus, Grafana...
│   ├── Frontend/          ← React, Angular, NextJS, Vite, TailwindCSS...
│   ├── Languages/         ← TypeScript, Python, Go, Java, Rust, Kotlin...
│   ├── ORM/               ← DrizzleORM, SpringDataJPA
│   ├── Testing/           ← Vitest, Playwright, Pytest, JUnit
│   └── Tools/             ← ESLint, Prettier, Zod, Swagger, Drizzle-Kit
├── Proyectos/             ← One entity page per project (state, stack, decisions)
├── Setup/                 ← Infrastructure, devices, services (Red-Local-Servicios.md is the network/ports source of truth)
├── Bitacora/              ← Agent work log — written ONLY through the `logbook` CLI, never by hand
├── AI/                    ← General AI notes
└── Temp/                  ← Drafts / temporary notes
```

`graphify-out/` also exists at the root: generated output, do not edit.

## Wikilink Rules — CRITICAL

Obsidian resolves wikilinks by **filename** (no path, no `.md`).

```markdown
[[NoteName]]                          ← simple link
[[NoteName|Visible text]]             ← link with alias
[[ExplicitFolder/NoteName|Alias]]     ← link with path (for disambiguation)
```

### Mandatory Rules

1. **Every document must have at least 2 wikilinks** to other vault documents
2. **Links must be bidirectional**: if A links to B, B must link to A
3. **"## See Also" section** at the end of EVERY note — always
4. **Stack notes**: include "## Related Stack" with sibling technologies
5. **Proyectos**: link to the Stack technologies they use
6. **Setup**: cross-link each other (Dispositivos ↔ Red-Local-Servicios ↔ relevant Proyectos)
7. **`Stack/_INDEX.md`**: update when a new note is added (new projects go in its "Proyectos documentados" table)
8. **Inside Markdown tables, do not put a wikilink with an alias** (`[[Note|alias]]` contains a `|` that breaks the table): link without alias or escape it as `\|`
9. **Match the note you edit**: keep its language and its field names (`**Última actualización:**` vs `**Last updated:**`) instead of mixing both

### Exact Filenames (for correct wikilinks)

Filenames without extension are the identifiers. **Do not trust a hardcoded list** (it drifts as the vault grows): list the real names before linking.

```bash
cd /mnt/nas/Obsidian
ls Stack/*/ Setup Proyectos          # names to use in [[...]]
test -f "Proyectos/<name>.md" && echo exists   # check one target
```

Prefer the path form for disambiguation (`[[Proyectos/video-studio|video-studio]]`, `[[Setup/Spark-IA|Spark-IA]]`, `[[Stack/AI-ML/ComfyUI|ComfyUI]]`). Note that `Proyectos/comfyui.md` (project) and `Stack/AI-ML/ComfyUI.md` (technology) coexist: the path form avoids linking the wrong one.

## Work Protocol

### Step 1 — Discover Current State
Local filesystem, read-only:
```bash
cd /mnt/nas/Obsidian
find . -name '*.md' -not -path './Bitacora/*' -not -path './graphify-out/*' | sort
grep -rl '\[\[' Setup/ --include='*.md'
```

### Step 2 — Audit Missing Links
For each note to create/update, identify:
- Which other vault notes mention it or should mention it?
- Does it have a "## See Also" section?
- Is it in `_INDEX.md`?

### Step 3 — Write/Update Notes
Write directly under `/mnt/nas/Obsidian/` with the file tools (Write / Edit), no SSH and no `sudo`:
- **New note:** create the file, but first check it does not exist (never overwrite a note you have not read).
- **Existing note:** edit only the section you need. Appending a `## See Also` block at the end is the usual back-link; keep the rest untouched.
- If you replace text with a script, assert the anchor appears exactly once before replacing.

### Step 4 — Update Notes That Should Link to the New One
If you create `Proyectos/new-tool.md` that uses React and TypeScript:
- Update `Stack/Frontend/React.md` → add to "## Projects Using It"
- Update `Stack/Languages/TypeScript.md` → same
- Update `Stack/_INDEX.md` → add a row in "Proyectos documentados"
- Add a `## See Also` back-link in any related `Setup/` or `Proyectos/` note (e.g. the machine's note)

### Step 5 — Verify
Check that every wikilink target exists and that the back-links are in place:
```bash
cd /mnt/nas/Obsidian
for l in $(grep -o '\[\[[^]|]*' Proyectos/new-tool.md | sed 's/\[\[//' | sort -u); do
  [ -f "$l.md" ] || [ -n "$(find . -name "$(basename "$l").md" | head -1)" ] && echo "OK  $l" || echo "MISSING  $l"
done
grep -l "new-tool" Stack/_INDEX.md Setup/*.md Proyectos/*.md      # back-links
```
Optional: the app reads the same vault through `obsidian-api` (maya:20000), so `curl -s http://atlas.casa/api/notes` should return 200 and list the new note.

## Templates

### Template: Stack Note

```markdown
# {Name}

> {One-line description}. Used with [[{Related1}]] and [[{Related2}]].

## What Is It?
{2-3 line description. Include at least 2 wikilinks to related technologies.}

## Related Stack
[[Tech1]] · [[Tech2]] · [[Tech3]]

## Installation
```bash
{install command}
```

## Basic Usage
```{lang}
{minimal working example}
```

## Projects Using It
- [[Proyectos/project-name|Project Name]] — {how it is used}

## See Also
- [[RelatedTechnology1]] — {relationship}
- [[RelatedTechnology2]] — {relationship}
- [[Red-Local-Servicios]] — {only if it applies to the intranet}
```

### Template: Project Note

```markdown
# {Project Name}

**Last updated:** {YYYY-MM-DD}

## Description
{What the project does. Link the stack technologies used.}

## Stack

| Layer | Technology |
|-------|-----------|
| Frontend | [[React]], [[Vite]], [[TypeScript]] |
| Backend | [[ExpressJS\|Express]], [[TypeScript]] |
| ...

## Architecture
```
{ASCII diagram or description}
```

## Project State
- [x] Feature 1
- [ ] Feature 2 (pending)

## Deploy
```bash
{deploy commands}
```

## See Also
- [[Red-Local-Servicios]] — nginx configuration
- [[Dispositivos]] — machines involved
- [[Related Stack]] — technologies used
```

### Template: Setup Note

```markdown
# {Service/Config}

**Last updated:** {YYYY-MM-DD}

## Description
{What it is and what it does on the intranet.}

## Infrastructure

| Data | Value |
|------|-------|
| Machine | {machine} ({IP}) — see [[Dispositivos]] |
| Domain | `service.casa` |
| ...

## Configuration
{details}

## Useful Commands
```bash
{commands}
```

## See Also
- [[Dispositivos]] — network inventory
- [[Red-Local-Servicios]] — nginx configuration
- [[Proyectos/related-project]] — {if applicable}
```

## Date and Format Conventions

- Date: `**Last updated:** YYYY-MM-DD`
- H1 heading required in every note
- Tables for structured data (inventories, endpoints, commands)
- Code blocks with language specified (` ```bash `, ` ```tsx `, etc.)
- Emojis optional in section headings (only in `_INDEX.md`)

## Current .casa Domains (for references in notes)

> Quick reference only. The source of truth for domains, ports and which nginx serves what is `Setup/Red-Local-Servicios.md`: check it before writing a machine or port into a note, and refresh this table when it changes. New Docker services on maya take ports in the `20000+` range (see that note).
>
> **Machine** = where the service itself runs. Which nginx fronts a domain (maya or pihole2) has changed over time and some rows in that note disagree; do not infer it from this table.

| Domain | Service | Machine |
|--------|---------|---------|
| `atlas.casa` | Notes app (Atlas) — `obsidian-api` on `:20000` | maya |
| `obsidian.casa` | Legacy Atlas alias | maya |
| `portal.casa` | Intranet dashboard (moved to maya's nginx on 2026-10-20) | maya |
| `ha.casa` | Home Assistant | pihole2 |
| `nas.casa` | FileBrowser NAS | maya |
| `maya.casa` | maya machine | maya |
| `pihole2.casa` | Pi-hole 2 panel | pihole2 |
| `pihole1.casa` | Pi-hole 1 panel | pihole1 |
| `clockwork.casa` | clockworkpi | clockwork |
| `router.casa` | Router | 192.168.1.1 |

## Model routing hints

- preferred agent: documenter
- preferred model: ollama/qwen3.6:27b
- routing intent: hint only; the skill must not switch models directly
