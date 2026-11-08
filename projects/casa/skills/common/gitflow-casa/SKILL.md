---
name: gitflow-casa
description: >
  Applies GitFlow correctly in any repository: feature/* → develop → release → main.
  Checks branch/state, detects violations, guides commits, merges, tags, and lefthook.
  Trigger: commit, merge, release, hooks, "which branch", "how do I push", "am I following gitflow?".
license: Apache-2.0
metadata:
  author: Felipe Perez
  version: "1.4"
---

## When to Use

- The user wants to commit, merge, publish, or tag a version.
- Asks which branch to use or whether the repo follows GitFlow.
- There are changes or commits on the wrong branch.
- Wants to configure or verify lefthook.

## Quick Protocol

1. Diagnose before touching anything:

```bash
git branch --show-current
git status --short
git log --oneline -5
git branch -a | grep -E "main|develop|feature|release|hotfix"
```

2. Apply the branch model:

```text
main      ← releases/hotfixes only, never direct commits
develop   ← integration
feature/* ← branches from develop, merges back to develop
release/* ← branches from develop, merges to main and develop
hotfix/*  ← branches from main, merges to main and develop
```

3. Starting new work:

```bash
git checkout develop
git pull origin develop
git checkout -b feature/<kebab-case-name>
```

4. Conventional commit:

```text
<type>(<scope>): <imperative description in lowercase, no trailing period>
```

Types: `feat`, `fix`, `refactor`, `perf`, `test`, `docs`, `style`, `chore`, `ci`, `revert`.

5. Releasing (release/hotfix on a GitHub remote): merge to `main` → tag `v<semver>` → merge back to `develop` → **create the GitHub Release as the last step**:

```bash
gh release create v<semver> --verify-tag --title "v<semver>" --notes "<changes>"
```

6. For the full release/hotfix commands (including the PR-based variant) or lefthook, load the full protocol.

## Detailed Reference

For complete release/hotfix commands, commit rules, `lefthook.yml`, hooks, and troubleshooting, load:

- `references/full-protocol.md`

## Rules

- ONLY use `pnpm` — never `npm` or `npx`. Use `pnpm dlx` instead of `npx`.
- Never commit directly to `main` or `develop`.
- If `develop` is missing, propose creating it from `main` before continuing.
- If there are uncommitted changes, do not switch branches without protecting them first.
- If you detect a GitFlow violation, explain the risk and propose the minimal fix.
- Do not skip hooks unless the user explicitly justifies it.
- A release is not finished until the GitHub Release exists (GitHub remotes only). Always `--verify-tag`, always after the tag is pushed and `main` is merged back into `develop`.
- Never push, open a PR or publish a release on a repo the user does not own without asking; name the target repo explicitly (`--repo`) when the clone has several remotes.
- Security by default: before committing, check the staged diff for secrets, keys or tokens and never commit them. Treat hooks as security gates, not obstacles.
