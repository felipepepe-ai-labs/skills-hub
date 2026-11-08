# Full Protocol

---
name: gitflow-casa
description: >
  Applies GitFlow correctly in any repository: feature/* → develop → release → main.
  Checks the current git state, detects flow violations, guides every step (branch,
  commit, merge, tag), and manages lefthook (install, pre-commit hooks: gitleaks,
  prettier, eslint). Trigger: the user wants to commit, merge, release, configure hooks,
  or asks "am I following gitflow?", "how do I push", "which branch do I use",
  "configure lefthook".
license: Apache-2.0
metadata:
  author: Felipe Perez
  version: "1.2"
---

## When to Use This Skill

- The user wants to commit, merge, or publish changes
- The user asks which branch they are on or which branch to use
- Changes are detected on the wrong branch (e.g. commits directly on `main`)
- Starting new work on a repo
- The user wants to configure or verify lefthook in a project

---

## Branch Model

```
main        ← only receives merges from release/* or hotfix/*. Never direct commits.
develop     ← integration. Receives merges from feature/* and hotfix/*.
feature/*   ← new work. Branches from develop, returns to develop.
release/*   ← version preparation. Branches from develop, merges to main AND develop.
hotfix/*    ← urgent production fix. Branches from main, merges to main AND develop.
```

**Critical rule**: Never run `git commit` directly on `main` or `develop`.

---

## Step 1 — Initial Diagnosis

```bash
git branch --show-current          # current branch
git status                         # pending changes
git log --oneline -5               # last commits
git branch -a | grep -E "main|develop|feature|release|hotfix"
```

Evaluate:
- Are we on the correct branch for the intended work?
- Are there commits on `main` or `develop` that should be on a `feature/*`?
- Does the `develop` branch exist? If not, create it from `main`.

---

## Step 2 — Common Flows

### New Work (feature)

```bash
git checkout develop
git pull origin develop            # sync before branching
git checkout -b feature/<name>     # kebab-case, descriptive name

# ... work, commits ...
git add <files>
git commit -m "feat(<scope>): description"

# When ready:
git checkout develop
git merge --no-ff feature/<name>
git push origin develop
git branch -d feature/<name>
```

### Publish a Version (release)

```bash
git checkout develop
git pull origin develop
git checkout -b release/<semver>   # e.g. release/1.2.0

# Version bumps, CHANGELOG, last fixes...
git commit -m "chore: bump version to <semver>"

# Merge to main
git checkout main
git merge --no-ff release/<semver>
git tag -a v<semver> -m "Release <semver>"
git push origin main --tags

# Merge back to develop
git checkout develop
git merge --no-ff release/<semver>
git push origin develop

git branch -d release/<semver>

# GitHub Release (only if the remote is on github.com; the tag must already be pushed)
gh release create v<semver> --verify-tag --title "v<semver>" --notes "<changes since the last tag>"
```

**GitHub Release rules**
- Run it LAST: after the tag is pushed and `main` is merged back into `develop`.
- Always `--verify-tag`: it fails instead of silently creating a new tag on the default branch.
- Notes: the commits/PRs since the previous tag, plus any known limitation. Use `--generate-notes` only if the repo has no curated notes.
- Target the right repo explicitly (`--repo <owner>/<repo>`) when the clone has several remotes; never publish to a repo the user does not own without asking.
- `gh release view v<semver> --json tagName,isDraft,isPrerelease` to verify.

#### PR-based variant (remote on GitHub)
When merges go through pull requests instead of local `git merge`:

```bash
git checkout -b release/<semver> develop && git push -u <remote> release/<semver>
gh pr create --base main --head release/<semver> --title "release: <semver>"
gh pr merge <n> --merge                      # only when the user asked to merge
git fetch <remote> && git checkout main && git merge --ff-only <remote>/main
git tag -a v<semver> -m "release <semver>" && git push <remote> v<semver>
git checkout develop && git merge --no-edit main && git push <remote> develop   # back-merge
gh release create v<semver> --verify-tag ...    # last step
```

### Urgent Fix (hotfix)

```bash
git checkout main
git pull origin main
git checkout -b hotfix/<description>

# Fix + commit
git commit -m "fix(<scope>): description of the fix"

# Merge to main
git checkout main
git merge --no-ff hotfix/<description>
git tag -a v<semver-patch> -m "Hotfix <semver-patch>"
git push origin main --tags

# Merge to develop
git checkout develop
git merge --no-ff hotfix/<description>
git push origin develop

git branch -d hotfix/<description>

# GitHub Release (same rules as for a release: last step, --verify-tag)
gh release create v<semver-patch> --verify-tag --title "v<semver-patch>" --notes "<the fix>"
```

---

## Commit Message Convention

### Full Format

```
<type>(<scope>): <description>        ← subject: max 72 characters

<body>                                ← optional, separated by blank line

<footer>                              ← optional: breaking changes, issues
```

### Subject Rules (mandatory)

| Rule | Correct | Incorrect |
|------|---------|-----------|
| Imperative present tense | `add login page` | `added login page` / `adds login page` |
| Lowercase | `fix cors header` | `Fix CORS header` |
| No trailing period | `refactor auth module` | `refactor auth module.` |
| Max 72 characters | — | long lines make `git log --oneline` hard to read |
| Scope in lowercase kebab-case | `feat(board-detail)` | `feat(BoardDetail)` |

### Types

| Type | When to use |
|------|-------------|
| `feat` | New user-facing functionality |
| `fix` | Bug fix |
| `refactor` | Internal change without altering behavior or adding features |
| `perf` | Performance improvement |
| `test` | Add or fix tests |
| `docs` | Documentation only (CLAUDE.md, README, comments) |
| `style` | Formatting, spaces, commas — no logic change |
| `chore` | Maintenance tasks: version bump, config, dependencies |
| `ci` | CI/CD pipeline changes |
| `revert` | Revert a prior commit |

### Body (when to write it)

Write a body when the subject alone is not enough to understand the **why**:
- The decision is not obvious
- Relevant alternatives were discarded
- There is business or technical context not visible in the code

```
refactor(db): replace in-memory store with drizzle + postgresql

The in-memory store was losing all data on backend restart, making
the app unusable across sessions. Drizzle was chosen over Prisma for
its lightweight query builder and zero-codegen approach.

Closes #12
```

### Footer

```
BREAKING CHANGE: <description of the incompatible change>
Closes #<issue>
Co-authored-by: Name <email>
```

### Full Examples

```
feat(auth): add TOTP MFA as optional second factor
```

```
fix(api): prevent 500 on missing board id in url params

Express was passing undefined to the repository when :boardId was
omitted. Added early validation in the controller before DB call.
```

```
chore: bump version to 1.2.0
```

```
refactor(lists): extract list ordering into dedicated service

BREAKING CHANGE: GET /api/boards/:id now returns lists sorted by
position field instead of insertion order.
```

```
feat(board-detail): implement CDK drag-and-drop for card reordering

Closes #34
```

### What is NOT a good commit message

```
❌  fix stuff
❌  WIP
❌  various changes
❌  fixed yesterday's bug
❌  feat: implemented the new feature for the drag and drop functionality in the board detail component
```

---

## Violation Detection

If any of these situations are detected, warn before continuing:

| Situation | Risk | Action |
|-----------|------|--------|
| Direct commits on `main` | Breaks release history | Create `hotfix/*` branch and cherry-pick |
| Direct commits on `develop` | Makes rollback harder | Create retroactive `feature/*` if applicable |
| `feature/*` far behind `develop` | Conflicts on merge | `git rebase develop` or `git merge develop` |
| No `develop` branch | No real gitflow | Create `develop` from current `main` commit |
| Version tag on non-`main` branch | Version not traceable | Move the tag after merging to `main` |
| Tag pushed but no GitHub Release (GitHub remote) | Release not visible/downloadable | `gh release create v<semver> --verify-tag ...` |
| GitHub Release created before the tag or before merging to `main` | Release points at the wrong commit | Delete the release, fix the tag, recreate |

---

## Lefthook — Pre-commit Hooks

### Diagnosis

```bash
# Is lefthook installed in the repo?
ls .git/hooks/pre-commit 2>/dev/null && echo "hooks installed" || echo "no hooks"
cat lefthook.yml 2>/dev/null || echo "no lefthook.yml"
```

### Installation in a New Repo

```bash
# 1. Add as devDependency
pnpm add -D lefthook

# 2. Create lefthook.yml in the repo root
# 3. Install hooks in .git/hooks/
pnpm lefthook install
```

### Standard `lefthook.yml` for casa projects

```yaml
pre-commit:
  parallel: true
  commands:
    gitleaks:
      run: gitleaks protect --staged --redact

    prettier:
      glob: "*.{ts,html,scss,json,md}"
      run: pnpm dlx prettier --write {staged_files}
      stage_fixed: true

    lint-backend:
      root: backend/
      glob: "src/**/*.ts"
      run: pnpm run lint -- --max-warnings 0

    lint-frontend:
      root: frontend/
      glob: "src/**/*.ts"
      run: node_modules/.bin/ng lint --quiet
```

**Adjust for the project:**
- Frontend only (no backend): remove `lint-backend`
- Backend only (no Angular): change `lint-frontend` to `pnpm run lint`
- No global prettier: move prettier under each `root` with its own config

### Available Hooks

| Hook | When it runs | Typical use |
|------|-------------|-------------|
| `pre-commit` | Before each commit | lint, format, secrets scan |
| `commit-msg` | When writing the message | Validate conventional commit format |
| `pre-push` | Before push | Tests, build check |

### `commit-msg` for Conventional Commits (recommended)

```yaml
commit-msg:
  commands:
    validate:
      run: |
        MSG=$(head -1 {1})
        # Valid type, optional scope in kebab-case, lowercase description, max 72 chars, no trailing period
        echo "$MSG" | grep -qP "^(feat|fix|refactor|perf|test|docs|style|chore|ci|revert)(\([a-z0-9-]+\))?: [a-z].{0,69}[^.]$" \
          || (echo "
        ❌ Commit rejected. Required format:
           type(scope): imperative description, lowercase, max 72 chars, no trailing period

           Types: feat | fix | refactor | perf | test | docs | style | chore | ci | revert
           Example: feat(auth): add TOTP MFA support
        " && exit 1)
```

### Useful Commands

```bash
pnpm lefthook install          # install/reinstall hooks
pnpm lefthook run pre-commit   # run hooks manually without committing
pnpm lefthook uninstall        # remove hooks from .git/hooks/

# Skip hooks temporarily (only if there is a very good reason):
git commit --no-verify -m "..."   # ⚠️ use with care
```

### Common Troubleshooting

| Problem | Cause | Solution |
|---------|-------|---------|
| Hook does not run | `lefthook install` was not run | `pnpm lefthook install` |
| `ng: not found` in frontend hook | ng not in hook PATH | Use `node_modules/.bin/ng` |
| Prettier reformats and commit fails | `stage_fixed: true` is missing | Add `stage_fixed: true` to the prettier command |
| gitleaks fails because it is not installed | Binary missing | `brew install gitleaks` or `apt install gitleaks` |

---

## Known Repos with GitFlow

| Repo | Path | Main branches |
|------|------|--------------|
| poc-trello | `/mnt/nas/sources/poc-trello` | main, develop |
| engram | `/mnt/nas/sources/engram` | main, develop |
| openclaw | `/mnt/nas/sources/openclaw` | main, develop |
| skills-hub | `/mnt/nas/sources/skills-hub` | main, develop |
| sdd-office | `/mnt/nas/sources/sdd-office` | main, develop |
