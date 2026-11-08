# Contributing Guide

Gracias por contribuir a skills-hub.

## Objetivo

Mantener este repositorio como fuente unica de verdad para sincronizar skills y prompts a destinos locales.

## Reglas clave

- Cambios pequenos y explicitos.
- No hardcodear rutas fuera de `config/sync-map.sh`.
- Mantener scripts Bash con `set -euo pipefail`.
- Evitar cambios destructivos no intencionales con `rsync --delete`.
- Mantener cada `SKILL.md` por debajo de 300 lineas; si crece mas, modularizar en `references/`.
- No introducir naming obsoleto (`mente`, `mente.casa`) cuando la convención actual sea `atlas`, `atlas.casa`.
- En skills JS/TS nuevas, preferir `pnpm` y documentar `minimumReleaseAge` cuando la skill cubra setup/bootstrap.
- Si se cambia la logica de instalacion, mantener alineados `bin/skills-hub.js` y `scripts/sync.sh`.
- Las skills se instalan por copia (rsync), nunca por symlink; ni el clon ni los destinos pueden vivir en un NAS.
- Una skill del intranet va a `projects/casa/skills/common`; una skill portable va a `projects/workflow/skills/common`.
- Tratar `projects/` como fuente canonica y las rutas de apps como destinos de exposicion (copias).
- Evitar colisiones de nombre entre skills expuestas a una misma app.

## Flujo de trabajo

1. Crea una rama desde `main`.
2. Realiza cambios acotados.
3. Ejecuta validaciones locales:

```bash
./scripts/doctor-skills.sh
./scripts/doctor.sh
./scripts/lint.sh
./scripts/check.sh
```

4. Actualiza documentacion si cambias convenciones o estructura.
5. Abre un Pull Request usando la plantilla.

## GitFlow obligatorio

- `feature/*` nace desde `develop` y hace PR a `develop`
- `release/vX.Y.Z` nace desde `develop` y hace PR a `main`
- `hotfix/*` nace desde `main` y hace PR a `main`
- no abrir PRs arbitrarios entre ramas fuera de esa matriz

Ver:

- `GITFLOW.md`
- `BRANCH_PROTECTION.md`
- `RELEASING.md`

## Convenciones de cambios

- `projects/casa/`: skills específicas del intranet `.casa`.
- `projects/workflow/`: skills portables de desarrollo y operación.
- Dentro de cada proyecto: `skills/common`, `skills/copilot-only` y `skills/claude-only`.
- `prompts/`: prompts e instrucciones para VS Code.
- `sdd-propose` es la skill canónica para propuestas SDD; `sdd-proposal` queda solo como alias legacy de compatibilidad.
- `projects/workflow/skills/common/skills-catalog-maintainer` es la referencia para auditar y refactorizar el propio catálogo.

## Checklist rapido antes de PR

- [ ] `./scripts/lint.sh` pasa sin errores.
- [ ] `./scripts/doctor.sh` pasa sin errores criticos.
- [ ] `./scripts/doctor-skills.sh` pasa sin errores.
- [ ] `./scripts/check.sh` pasa en entorno local.
- [ ] No se introdujeron rutas hardcodeadas en scripts.
- [ ] No se introdujeron referencias obsoletas a `mente.casa` o `mente-docs`.
- [ ] README y/o instrucciones actualizadas si aplica.

## Seguridad y ownership

- Para hallazgos sensibles, usar el proceso de `SECURITY.md` en lugar de issue publico.
- Configurar `.github/CODEOWNERS` con los handles reales antes de exigir revisiones obligatorias.
