# Skills Catalog Maintenance Checklist

- confirm the skill belongs in `common` vs platform-specific sources
- confirm folder name equals frontmatter `name`
- keep `SKILL.md` under 300 lines
- move long templates/examples/tables to `references/` or `assets/`
- remove stale naming in docs, prompts, and managed config
- verify no duplicate skill names are exposed to the same app
- run `./scripts/validate-skills.sh`
- run `./scripts/doctor-skills.sh`
- run `./scripts/lint.sh`
- run `./scripts/doctor.sh`
