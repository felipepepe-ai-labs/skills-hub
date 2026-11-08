---
name: poc-init
description: >
  Initializes a new PoC project with the standard stack: Angular 21 +
  Express/TypeScript + Drizzle ORM + OpenAPI + GitFlow + CLAUDE.md + structure
  ready for deploy-casa. Trigger: "create a new poc", "new project",
  "initialize project X", "scaffolding for X".
license: Apache-2.0
metadata:
  author: Felipe Perez
  version: "1.1"
---

## When to Use

- The user wants to start a new project from scratch.
- They ask for a PoC, scaffold, or standard project for the `.casa` intranet.
- A frontend+backend base is needed without reinventing structure.

## Standard Stack

| Layer | Technology |
|-------|------------|
| Frontend | Angular 21 + Angular Material/CDK |
| State | Signals + RxJS |
| Backend | Express + TypeScript |
| ORM/DB | Drizzle ORM + PostgreSQL |
| API | OpenAPI 3.0 |
| Tests | Vitest |
| Quality | ESLint + Prettier + Lefthook |
| Deploy | compatible with `deploy-casa` |

## Quick Protocol

1. Confirm project name, desired `.casa` domain, and whether it needs auth.
2. Create the repo with this structure:

```text
<project>/
├── backend/
├── frontend/
├── .github/copilot-instructions.md
├── CLAUDE.md
├── lefthook.yml
└── start-dev.sh
```

3. Initialize GitFlow:

```bash
git init
git checkout -b main
git checkout -b develop
git checkout -b feature/init
```

4. Generate `.env.example`, never real secrets.
5. Add OpenAPI, minimum tests, and dev/build/test scripts.
6. Validate that both backend and frontend start before marking it complete.

## Detailed Reference

For the full folder tree, `package.json` examples, OpenAPI, Drizzle, Angular, scripts, and final checklist, load:

- `references/full-protocol.md`

## Rules

- Do not create real secrets in `.env`; use `.env.example` or Infisical if applicable.
- Keep TypeScript strict in both backend and frontend.
- Use functional Express controllers, not classes, unless the project requires otherwise.
- Every PoC must ship with minimum tests and a working `start-dev.sh`.
- If the user wants something smaller than the standard stack, explicitly reduce the scope.
- Ship secure defaults from the first scaffold: validate input, apply least privilege, keep secrets out of code (use Infisical). Security is a design input, not a later review step.
