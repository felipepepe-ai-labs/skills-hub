---
name: skills-catalog-maintainer
description: >
  Use when auditing, refactoring, renaming, modularizing, or validating a shared skills catalog that is exposed to multiple agent platforms.
license: Apache-2.0
metadata:
  author: Felipe Pérez
  version: "1.0"
---

## When to Use

- A skill repo is the source of truth for several agent platforms
- You are adding, renaming, or deleting skills
- You need to enforce naming, modularity, or exposure rules
- You want to prevent legacy references, duplicate skills, or broken catalog structure

## Quick Decision

| Situation | Action |
|---|---|
| New reusable skill | Place it in `skills/common` if platform-agnostic |
| Platform-specific behavior | Keep it in `skills/copilot-only` or equivalent |
| SKILL grows too much | Split support material into `references/` or `assets/` |
| Renaming a skill | Update folder, frontmatter, docs, and routing references together |
| Catalog uncertainty | Run the doctor and fix findings before continuing |

## Critical Patterns

- Treat the repository as the **canonical source**
- Agent install paths are **exposure targets**, not authoring locations
- Folder name and frontmatter `name` must match exactly
- Avoid duplicate skill names across sources exposed to the same app
- Keep `SKILL.md` concise; move templates, tables, or long examples out
- Kill legacy naming fast; do not let aliases become parallel standards

## Maintenance Workflow

### 1. Classify correctly

- `skills/common` for portable skills
- `skills/copilot-only` for Copilot/OpenCode specific workflows
- keep support content near the skill in `references/` or `assets/`

### 2. Preserve canonical exposure

Before finishing a change:
- check that `config/apps.json` still points to the right source directories
- verify there is no duplicate skill name inside any app exposure set
- validate install/sync tooling still sees the catalog correctly

### 3. Enforce editorial consistency

Every skill should keep:
- aligned folder/frontmatter name
- clear trigger-oriented description
- concise protocol
- support material extracted when needed

### 4. Clean legacy references

After any rename or canonical change:
- search docs, scripts, prompts, and managed config
- replace obsolete names in examples and routing text
- keep aliases only when strict backward compatibility is required

## Commands

```bash
./scripts/validate-skills.sh
./scripts/doctor-skills.sh
./scripts/lint.sh
./scripts/doctor.sh
```

## Resources

- `references/checklist.md`
- `references/canonical-exposure.md`
