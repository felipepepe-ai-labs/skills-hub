---
name: infisical-admin-mcp
description: >
  Provisions projects and bootstrap credentials in Infisical using the global
  `infisical-admin` MCP. Trigger: when the user asks to create a project/vault in
  Infisical, get `clientId` and `clientSecret`, create or reuse machine identities,
  initialize Universal Auth, or automate Infisical administration tasks without
  going through the UI manually.
license: Apache-2.0
metadata:
  author: Felipe Pérez
  version: "1.0"
---

Use the global MCP `infisical-admin` as the preferred path for provisioning operations
in Infisical. Do not go to the UI or manual flows first if the MCP covers the task.

## When to Use This Skill

- The user asks for `clientId` and `clientSecret`
- The user wants to create or initialize a project/vault in Infisical
- The user wants to create or reuse a machine identity
- The user wants to automate Universal Auth or bootstrap credentials

## Preferred MCP Tools

- `infisical_list_projects`
- `infisical_bootstrap_project`
- `infisical_ensure_credentials`

## Recommended Flow

1. If it is not clear whether the project exists, use `infisical_list_projects`.
2. If the user wants to create or set up everything from scratch, use `infisical_bootstrap_project`.
3. If the project already exists and only credentials are missing, use `infisical_ensure_credentials`.
4. Return `projectId`, `identityId`, `clientId`, and `clientSecret` to the user when applicable.

## Rules

- Treat `clientSecret` as sensitive data.
- Do not write real secrets to `.env` except for bootstrap credentials explicitly permitted by the project.
- If another Infisical skill also applies, use this skill first for provisioning and then the other for code integration.
