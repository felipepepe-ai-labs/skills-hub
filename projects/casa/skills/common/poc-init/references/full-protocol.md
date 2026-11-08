# Full Protocol

---
name: poc-init
description: >
  Inicializa un nuevo proyecto PoC con el stack estándar: Angular 21 +
  Express/TypeScript + Drizzle ORM + OpenAPI + gitflow + CLAUDE.md +
  estructura lista para deploy-casa. Trigger: "crea un nuevo poc", "nuevo
  proyecto", "inicializa el proyecto X", "scaffolding para X".
license: Apache-2.0
metadata:
  author: Felipe Perez
  version: "1.0"
---

## Cuándo usar este skill

- El usuario quiere arrancar un proyecto nuevo desde cero
- Pide "nuevo PoC", "crea el proyecto X", "scaffolding"
- Hay que montar la base (frontend + backend) sin reinventar la rueda

---

## Stack estándar

| Capa | Tecnología | Versión |
|------|-----------|---------|
| Frontend | Angular | 21.x |
| UI | Angular Material + CDK | 21.x |
| Estado | Signals + RxJS | — |
| Build | esbuild (`@angular/build:application`) | — |
| Backend | Express + TypeScript | 4.x / 5.x |
| ORM | Drizzle ORM | latest |
| DB | PostgreSQL | — |
| Auth | JWT + bcryptjs | — |
| API spec | OpenAPI 3.0 (YAML) | — |
| Tests | Vitest | — |
| Linting | ESLint + Prettier | — |
| Hooks | Lefthook | — |

---

## Estructura de directorios

```
<proyecto>/
├── backend/
│   ├── src/
│   │   ├── controllers/     # handlers funcionales (no clases)
│   │   ├── routes/          # Express routers
│   │   ├── db/
│   │   │   ├── schema.ts    # Drizzle table definitions
│   │   │   └── repositories/
│   │   ├── models/          # interfaces de dominio + DTOs
│   │   ├── openapi/         # openapi.yaml
│   │   ├── app.ts           # Express setup, middleware, Swagger UI
│   │   └── server.ts        # entry point (escucha puerto)
│   ├── migrations/
│   ├── .env.example
│   ├── drizzle.config.ts
│   ├── tsconfig.json        # strict: true, rootDir: src, outDir: dist
│   ├── vitest.config.ts
│   └── package.json
│
├── frontend/
│   ├── src/app/
│   │   ├── features/        # componentes standalone por feature
│   │   ├── services/        # HttpClient wrappers
│   │   ├── core/            # guards, interceptors
│   │   └── models/          # interfaces espejo del backend
│   ├── proxy.conf.json      # /api → localhost:3000
│   ├── tsconfig.json        # strict: true, strictTemplates: true
│   └── package.json
│
├── .github/
│   └── copilot-instructions.md
├── lefthook.yml
├── start-dev.sh
└── CLAUDE.md
```

---

## Paso 1 — Preguntas antes de generar

1. **Nombre del proyecto** (kebab-case, ej: `poc-inventario`)
2. **Descripción** en una línea
3. **¿Necesita auth?** (JWT básico / JWT + MFA / sin auth)
4. **¿Puerto del backend?** (ver tabla de puertos libres en maya)
5. **¿Dominio `.casa`?** (para configurar CORS_ORIGIN y planificar deploy)

---

## Paso 2 — Crear estructura base

```bash
BASE=/mnt/nas/sources/<proyecto>
mkdir -p $BASE/{backend/src/{controllers,routes,db/repositories,models,openapi},backend/migrations,frontend}
cd $BASE
```

---

## Paso 3 — Backend: ficheros esenciales

### `backend/src/server.ts`
```typescript
import app from './app';
import { env } from './env';

const PORT = env.PORT;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### `backend/src/app.ts`
```typescript
import express from 'express';
import cors from 'cors';
import swaggerUi from 'swagger-ui-express';
import YAML from 'yamljs';
import path from 'path';
import { env } from './env';

const app = express();
const spec = YAML.load(path.join(__dirname, 'openapi/openapi.yaml'));

app.use(cors({ origin: env.CORS_ORIGIN }));
app.use(express.json());
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(spec));
app.get('/health', (_req, res) => res.json({ status: 'ok', timestamp: new Date().toISOString() }));

// routes aquí

export default app;
```

### `backend/src/env.ts`
```typescript
import 'dotenv/config';

export const env = {
  PORT: parseInt(process.env.PORT ?? '3000'),
  DATABASE_URL: process.env.DATABASE_URL!,
  JWT_SECRET: process.env.JWT_SECRET!,
  JWT_EXPIRY: process.env.JWT_EXPIRY ?? '24h',
  CORS_ORIGIN: process.env.CORS_ORIGIN ?? 'http://localhost:4200',
};
```

### `backend/.env.example`
```
DATABASE_URL=postgresql://user:password@localhost:5432/<proyecto>
PORT=3000
CORS_ORIGIN=http://localhost:4200
JWT_SECRET=change-me
JWT_EXPIRY=24h
```

### `backend/tsconfig.json`
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "strict": true,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "esModuleInterop": true,
    "resolveJsonModule": true
  },
  "include": ["src"]
}
```

### `backend/drizzle.config.ts`
```typescript
import { defineConfig } from 'drizzle-kit';
import 'dotenv/config';

export default defineConfig({
  schema: './src/db/schema.ts',
  out: './migrations',
  dialect: 'postgresql',
  dbCredentials: { url: process.env.DATABASE_URL! },
});
```

### `backend/package.json` (scripts mínimos)
```json
{
  "scripts": {
    "dev": "ts-node-dev --respawn src/server.ts",
    "build": "tsc && cp -r src/openapi dist/src/openapi",
    "start": "node dist/src/server.js",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "lint": "eslint src/",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate"
  }
}
```

> ⚠️ El script `build` debe incluir `cp -r src/openapi dist/src/openapi` — tsc no copia assets YAML.

---

## Paso 4 — Frontend: configuración Angular

```bash
cd /mnt/nas/sources/<proyecto>
npx @angular/cli@21 new frontend \
  --routing \
  --style scss \
  --standalone \
  --skip-git \
  --package-manager npm
```

### `frontend/proxy.conf.json`
```json
{
  "/api": {
    "target": "http://localhost:3000",
    "secure": false
  }
}
```

Añadir al `angular.json` en `serve.options`: `"proxyConfig": "proxy.conf.json"`

### Convenciones Angular obligatorias
- **Siempre standalone** — sin NgModules
- **`inject()`** en cuerpo de clase (no constructor)
- **`signal()` / `computed()`** para estado local
- **RxJS** para llamadas HTTP (`.subscribe({ next, error })`)
- `inlineStyleLanguage: 'scss'`
- `strictTemplates: true` en tsconfig

---

## Paso 5 — Lefthook + linting

### `lefthook.yml`
```yaml
pre-commit:
  parallel: true
  commands:
    gitleaks:
      run: gitleaks protect --staged --redact
    prettier:
      glob: "*.{ts,html,scss,json}"
      run: npx prettier --write {staged_files}
    lint-backend:
      root: backend/
      glob: "src/**/*.ts"
      run: npm run lint -- --max-warnings 0
    lint-frontend:
      root: frontend/
      run: ng lint --quiet
```

```bash
cd /mnt/nas/sources/<proyecto> && npx lefthook install
```

---

## Paso 6 — CLAUDE.md

Generar con `/init` una vez creada la estructura, o usar como base el CLAUDE.md de `poc-trello` (`/mnt/nas/sources/poc-trello/CLAUDE.md`) adaptando nombres y puertos.

---

## Paso 7 — Gitflow inicial

```bash
cd /mnt/nas/sources/<proyecto>
git init
git checkout -b develop
git add .
git commit -m "chore: initial scaffolding"
git checkout -b main
git merge develop
git checkout develop   # volver a develop para trabajar
```

---

## Paso 8 — Base de datos local

```bash
sudo -u postgres psql -c "CREATE USER <proyecto> WITH PASSWORD '<proyecto>_local_$(date +%Y)';"
sudo -u postgres psql -c "CREATE DATABASE <proyecto> OWNER <proyecto>;"

# Actualizar .env con la DATABASE_URL real
cd /mnt/nas/sources/<proyecto>/backend
npm run db:migrate
```

---

## Paso 9 — Servicio systemd (para cuando esté listo para deploy)

Plantilla en `/etc/systemd/system/<proyecto>.service`:

```ini
[Unit]
Description=<proyecto> backend
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=felipe
WorkingDirectory=/mnt/nas/sources/<proyecto>/backend
EnvironmentFile=/mnt/nas/sources/<proyecto>/backend/.env
ExecStart=/home/felipe/.nvm/versions/node/v22.22.0/bin/node dist/src/server.js
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## Checklist final

- [ ] Estructura de directorios creada
- [ ] Backend: `server.ts`, `app.ts`, `env.ts`, `tsconfig.json`, `drizzle.config.ts`
- [ ] Backend: `.env.example` con todas las variables
- [ ] Backend: script `build` incluye copia de `openapi/`
- [ ] Frontend: Angular standalone, proxy.conf.json, strictTemplates
- [ ] Lefthook instalado
- [ ] CLAUDE.md generado
- [ ] Git init con rama `develop` como rama de trabajo
- [ ] Base de datos PostgreSQL creada y migraciones ejecutadas
- [ ] `start-dev.sh` con backend + frontend en paralelo
