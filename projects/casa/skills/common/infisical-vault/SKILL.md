---
name: infisical-vault
description: >
  Secrets management rules for all .casa intranet projects. From now on NO .env files
  with real secrets are used. All secrets are managed in Infisical self-hosted at
  maya.casa:8888. Trigger: whenever working with secrets, environment variables,
  connection configuration (DB, JWT, API keys), or .env is mentioned in any project.
license: Apache-2.0
metadata:
  author: Felipe Pérez
  version: "1.1"
---
> **🏠 CASA LAYER** — Only use this skill when working in a `.casa` intranet project.
> **Detect context by**: presence of `.casa` file in project root, `*.casa` domain references (maya.casa, infisical.casa, pihole.casa), or user explicitly mentions "intranet" or "home project".
> **Work/external projects**: do NOT suggest or use this skill. Skip silently.

## Main Rule (CRITICAL)

> **NEVER create or edit `.env` with real secrets.** Always use Infisical.

If the user or context asks to create a `.env`, a `.env.local`, or to put a secret
in plain text in any file: **stop, explain the policy, and use Infisical**.

The only exceptions allowed in `.env` are the Infisical bootstrap credentials:
`INFISICAL_CLIENT_ID`, `INFISICAL_CLIENT_SECRET`, `INFISICAL_SITE_URL`, and `USE_DATABASE`.
Everything else (DATABASE_URL, JWT_SECRET, API keys…) goes in the vault.

---

## Vault Infrastructure

| Data | Value |
|------|-------|
| **URL** | `http://maya.casa:8888` |
| **Machine** | `maya.casa` — `192.168.1.55` |
| **Docker Compose** | `/mnt/nas/sources/infisical/docker-compose.yml` on `felipe@192.168.1.55` |
| **Admin** | `admin@bitwarden.casa` / see Infisical |
| **Admin SSH** | `felipe@192.168.1.55` (passwordless, SSH key) |

---

## How to Integrate (Node / TypeScript)

### 1. Install the SDK

```bash
pnpm add @infisical/sdk
```

### 2. Secrets Module (`src/secrets.ts`)

```typescript
import { InfisicalSDK } from '@infisical/sdk';
// ⚠️ v5 renamed InfisicalClient → InfisicalSDK. Do NOT use InfisicalClient.

export async function loadSecrets(): Promise<void> {
  const client = new InfisicalSDK({
    siteUrl: process.env.INFISICAL_SITE_URL ?? 'http://maya.casa:8888',
  });
  // Constructor does NOT accept auth — call .auth() separately
  await client.auth().universalAuth.login({
    clientId: process.env.INFISICAL_CLIENT_ID!,
    clientSecret: process.env.INFISICAL_CLIENT_SECRET!,
  });

  const projectId = 'YOUR_PROJECT_ID';
  const environment = 'dev';

  const secretNames = ['DATABASE_URL', 'JWT_SECRET', 'JWT_EXPIRY', 'PORT', 'CORS_ORIGIN'];
  for (const secretName of secretNames) {
    const { secretValue } = await client.secrets().getSecret({
      secretName, projectId, environment,
    });
    process.env[secretName] = secretValue;
  }
}
```

### 3. Start the Server Only After Secrets Are Loaded

```typescript
// server.ts — dynamic imports ensure env.ts (Zod) parses
// AFTER loadSecrets() has injected values into process.env
import 'dotenv/config';           // only loads bootstrap (.env)
import { loadSecrets } from './secrets';

await loadSecrets();
const [{ default: app }, { env }] = await Promise.all([
  import('./app'),                // dynamic: AFTER secrets
  import('./env'),                // Zod .parse(process.env) here
]);
app.listen(env.PORT);
```

### 4. Minimum Required Environment Variables (these do go in the process environment)

```dotenv
# .env — bootstrap only (this DOES go in .env, it contains no real secrets)
INFISICAL_SITE_URL=http://maya.casa:8888
INFISICAL_CLIENT_ID=<machine-identity-client-id>
INFISICAL_CLIENT_SECRET=<machine-identity-client-secret>
USE_DATABASE=true
NODE_ENV=development
```

> **How to get clientId:** In Infisical UI → Identity → Universal Auth →
> `identityUniversalAuth.clientId` — this is NOT the `identityId`, they are different.

---

## Secrets Managed by Infisical for bitwarden-clone

| Key | Description |
|-----|-------------|
| `DATABASE_URL` | PostgreSQL on maya.casa |
| `JWT_SECRET` | JWT signing key |
| `JWT_EXPIRY` | Token expiry (`24h`) |
| `MFA_ISSUER` | TOTP issuer (`bitwarden.casa`) |
| `PORT` | Express server port |
| `CORS_ORIGIN` | Allowed origin (`http://localhost:4200`) |

---

## Flow for Adding a New Secret

1. Open `http://maya.casa:8888` → project → environment
2. Add the secret with its key/value
3. Update the `Secrets` type in `src/secrets.ts` if using TypeScript
4. **Never** put it in `.env`, `app.yaml`, `docker-compose.yml`, or in code

---

## .env.example Still Exists

The `.env.example` file **is kept** in the repo as documentation of which keys exist,
but with example values (`change-me`, `your-value-here`).
It never contains real values.

## Model routing hints

- preferred agent: security
- preferred model: ollama/qwen3.6:27b
- routing intent: hint only; the skill must not switch models directly
