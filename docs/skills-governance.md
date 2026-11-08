# Skills Governance

`skills-hub` is the canonical catalog for assistant skills. The repository is split into two project catalogs and keeps flat skill directories inside each platform variant so every installer can expose predictable sources without category traversal or custom routing.

## Layout

- Casa/intranet skills go in `projects/casa/skills/common/<skill>/SKILL.md`.
- Portable workflow skills go in `projects/workflow/skills/common/<skill>/SKILL.md`.
- Copilot/OpenCode-specific skills go in the selected project's `skills/copilot-only/<skill>/SKILL.md`.
- Claude-specific skills go in the selected project's `skills/claude-only/<skill>/SKILL.md`.
- Do not create nested folders like `projects/workflow/skills/common/analysis/<skill>/SKILL.md`.

## Governance Skills

The baseline governance layer is:

- `core-token-efficient-skill-governor` — global token-efficiency rules.
- `core-token-efficient-command-output` — concise command-output rules.
- `core-repository-safety-rules` — local-only repository safety.
- `quality-skill-quality-gate` — PASS/WARN/FAIL validation checklist.

Existing `projects/workflow/skills/common/skills-catalog-maintainer` remains the catalog maintenance reference for naming, modularity, exposure rules, and legacy cleanup. The new governance skills complement it by adding token-efficiency and bounded-output requirements.

## Creating a New Token-Efficient Skill

1. Choose `projects/casa` for `.casa` infrastructure; otherwise use `projects/workflow`.
2. Default to the selected project's `skills/common`.
3. Use a flat prefixed folder name when helpful.
4. Match folder name and frontmatter `name` exactly.
5. Keep `description` under 160 characters.
6. Include language, dependency, workflow, tool, safety, output, explanation, token-efficiency, and success criteria sections.
7. Keep `SKILL.md` under 300 lines; move long examples into `references/`.
8. Run cheap validation before finishing.

## Auditing Existing Skills

Use `agents-audit-skills`, `quality-skill-quality-gate`, and `skills-catalog-maintainer`. Inspect only likely catalog paths and classify skills as:

- `PASS`: structurally valid and aligned with safety/token-efficiency rules.
- `WARN`: usable but missing some governance polish or has legacy overlap.
- `FAIL`: broken folder/frontmatter alignment, missing `SKILL.md`, over 300 lines, unsafe defaults, or obsolete naming.

### Current Audit Snapshot

- `PASS`: both project catalogs are validated by `doctor-skills` and exposed through `config/apps.json`, resolved by `config/projects.json`.
- `WARN`: some existing skills predate the new governance section template; platform-specific overrides are preserved intentionally.
- `FAIL`: none identified in the canonical skill tree during this governance-layer addition.

## Quality Gate

`quality-skill-quality-gate` checks folder/frontmatter alignment, short description, explicit trigger, language policy, bounded workflow, tool policy, safety policy, output format, explanation policy, token-efficiency rules, success criteria, line count, `pnpm` convention, and obsolete naming avoidance.

## Recommended Validation Commands

Use cheap commands first:

```bash
git status --short
./scripts/doctor-skills.sh
./scripts/doctor.sh
./scripts/lint.sh
./scripts/check.sh
pnpm skills-hub status
pnpm skills-hub doctor
pnpm skills-hub doctor-skills
pnpm skills-hub lint
pnpm skills-hub check
```

`check.sh` can report local target drift; do not modify app-local targets unless the user explicitly asks to sync.

## Safety Rules

- Do not commit, push, create PRs, delete files, install packages, or run destructive commands unless explicitly requested.
- Apply local repository changes only.
- Do not modify app-local target directories.
- Preserve copy-based installation; do not introduce symlink-based installation.
- Do not change installation logic unless required; if changed, keep `bin/skills-hub.js` and `scripts/sync.sh` aligned.

## Token-Efficiency Rules

- Keep context narrow.
- Inspect only files needed for the task.
- Avoid broad repository scans unless required.
- Keep output bounded.
- Avoid long explanations and full file dumps.
- Prefer concise command output.
- Prefer English for internal instructions.
- Disable or avoid unnecessary tools.
