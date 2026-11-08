---
name: project-workflow
description: >
  Complete workflow for creating and deploying projects on the intranet.
  From zero to production: SDD + GitFlow + Infisical + Deploy + DNS + Atlas.
  Trigger: when the user wants to start a new project or asks how we work.
license: Apache-2.0
metadata:
  author: Felipe Pérez
  version: "1.0"
---

## Complete Flow

```
PHASE 1 — STARTUP
  casa project init <name> [--secrets] [--github] [--template node|angular]

PHASE 2 — DESIGN
  /sdd-new <name>   →  proposal → spec → design → tasks

PHASE 3 — DEVELOPMENT
  GitFlow (`feature/*`, `release/*`, `hotfix/*`) + conventional commits
  /sdd-apply        →  implement tasks
  casa atlas add    →  document decisions DURING development

PHASE 4 — VERIFICATION
  /sdd-verify       →  validate against spec
  local tests
  smoke test in local staging (docker compose up in dev)

PHASE 5 — RELEASE
  PR develop → main + code review (even if self-review)
  semantic tag (v1.0.0) + CHANGELOG

PHASE 6 — PRODUCTION
  casa deploy <name>
  health check: curl http://localhost:<port>/health
  casa domain add <name>.casa <ip> --port <n> --portal --icon <emoji> --desc "<desc>"

PHASE 7 — CLOSE
  casa atlas add project "<name>" --file note.md   (update with prod URL)
  engram mem_save (architecture, decisions, gotchas)
  /sdd-archive <name>
```

## Phase 1 — `casa project init` in Detail

```bash
# Minimum (git + deploy directory only)
ssh -o BatchMode=yes felipe@192.168.1.55 'casa project init <name>'

# With Infisical vault (if the project uses secrets)
ssh -o BatchMode=yes felipe@192.168.1.55 'casa project init <name> --secrets --envs dev,prod'

# With private GitHub repo
ssh -o BatchMode=yes felipe@192.168.1.55 'casa project init <name> --secrets --github'
```

This creates:
- `/home/felipe/<name>/` — deploy directory on maya
- Git repo with `main` and `develop` branches
- (opt) Private GitHub repo
- (opt) Infisical project with environments and Machine Identity

## Phase 2 — SDD

```
/sdd-new <name>        ← starts the full cycle
/sdd-continue <name>   ← continues to the next phase
/sdd-apply <name>      ← implements the tasks
/sdd-verify <name>     ← validates against spec
/sdd-archive <name>    ← closes the change
```

## Phase 3 — GitFlow

```bash
# Feature
git checkout develop
git pull
git checkout -b feature/<name>

# Conventional commits
git commit -m "feat: add login endpoint"
git commit -m "fix: handle expired JWT"
git commit -m "chore: update dependencies"

# PR feature → develop
gh pr create --base develop --title "feat: <description>"

# Release
git checkout develop
git pull
git checkout -b release/v1.0.0
gh pr create --base main --title "release: v1.0.0"

# After merge to main
git tag v1.0.0
git checkout develop && git merge --no-ff main
```

## Phase 3 — Infisical (if `--secrets`)

```bash
# Output of casa project init --secrets:
# Client ID:     abc123
# Client Secret: xyz789   ← bootstrap .env only

# Minimal project .env (ONLY Infisical credentials)
INFISICAL_CLIENT_ID=abc123
INFISICAL_CLIENT_SECRET=xyz789
INFISICAL_SITE_URL=http://infisical.casa

# All other secrets go DIRECTLY into infisical.casa
# No more variables in .env
```

See skill `infisical-vault` for the code pattern.

## Phase 5 — Release Checklist

- [ ] All tests pass
- [ ] /sdd-verify green
- [ ] PR approved (even if self-review)
- [ ] CHANGELOG updated
- [ ] Semantic tag created
- [ ] `develop` updated with release changes

## Phase 6 — Deploy

```bash
# Deploy
ssh -o BatchMode=yes felipe@192.168.1.55 'casa deploy <name>'

# Post-deploy health check
ssh -o BatchMode=yes felipe@192.168.1.55 'curl -sf http://localhost:<port>/health || echo FAIL'

# Add domain + portal (only when health check passes)
ssh -o BatchMode=yes felipe@192.168.1.55 \
  'casa domain add <name>.casa 192.168.1.55 --port <port> --portal \
   --icon <emoji> --desc "<description>" --machine maya'

# Verify DNS from the network
ssh -o BatchMode=yes felipe@192.168.1.55 'casa domain list | grep <name>'
```

## Rollback

```bash
# If the deploy fails:
ssh -o BatchMode=yes felipe@192.168.1.55 \
  'cd /home/felipe/<name> && git checkout <previous-tag> && docker compose up -d'
```

## Phase 7 — Close Documentation

### Atlas (update note with prod URL)
```bash
ssh -o BatchMode=yes felipe@192.168.1.55 \
  "sed -i 's|## Deploy|## Production URL\n\`http://<name>.casa\`\n\n## Deploy|' \
  /mnt/nas/Obsidian/Projects/<name>.md"
```

### Engram
```
mem_save title: "Deployed <name> to production"
type: architecture
content: What/Why/Where/Learned
```

## Golden Rules

1. **Vault BEFORE code** — never `.env` with real data in the repo
2. **SDD BEFORE implementing** — spec and design first, code after
3. **Document DURING development** — not at the end
4. **Health check BEFORE the domain** — do not add to the portal if the service fails
5. **Conventional commits always** — feat/fix/chore/docs/refactor/test
6. **Security BY DEFAULT and BY DESIGN** — from init to deploy, not a release-time check: secure defaults, least privilege, validated input, no secrets in code

## Reference Tools

| Tool | Skill |
|------|-------|
| Full project | `project-workflow` (this) |
| .casa domains | — (no instalada) |
| Infisical vault | `infisical-vault` |
| Production deploy | — (no instalada) |
| Atlas documentation | `atlas-docs` |
| Full SDD | `sdd-*` skills |
| GitFlow | `gitflow` skill |

## Model routing hints

- preferred agent: architect
- preferred model: ollama/qwen3.6:27b
- routing intent: hint only; the skill must not switch models directly
