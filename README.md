# skills-hub

Fuente única de verdad para tus skills de asistentes de IA, pensada para **instalarlas en cualquiera de tus máquinas** desde un único repositorio en GitHub.

Incluye una CLI instalable que copia el catálogo a las apps locales compatibles.

## Objetivo

Mantener todas las skills versionadas en un solo repositorio y **distribuirlas a varias máquinas**: clonas el repo en disco local de cada equipo y copias las skills a los directorios de cada app.

- ~/.copilot/skills
- ~/.claude/skills
- ~/.agents/skills
- ~/.config/Code/User/prompts (contenido legacy)
- OpenCode (config gestionada)

### Principios

- **Fuente única en GitHub.** El repo es la verdad; cada máquina lo clona.
- **Todo local, sin NAS.** Ni el clon ni los destinos pueden vivir en un filesystem de red. Los scripts abortan si detectan NFS/CIFS/SMB/sshfs.
- **Copia, no symlinks.** Las skills se copian a cada app, de modo que quedan dentro de la máquina e independientes del clon. Tras editar una skill, re-ejecuta `sync` para propagar.

## Estructura

- `projects/casa`: proyecto con las skills específicas del intranet `.casa`.
- `projects/workflow`: proyecto con las skills portables de desarrollo y operación.
- Cada proyecto mantiene `skills/common`, `skills/copilot-only` y `skills/claude-only`.
- `prompts`: prompts globales e instrucciones (contenido copiable legacy).
- `config/apps.json`: manifiesto de apps detectables, sus rutas y qué fuentes consume cada una.
- `config/projects.json`: límites y fuentes de `casa` y `workflow`.
- `config/external-skills.json`: skills de terceros que el hub no versiona (origen, licencia y cómo reinstalarlas); ver `docs/external-skills.md`.
- `config/sync-map.sh`: mapeos legacy para contenido copiable como `prompts`.
- `opencode/opencode.managed.json`: fragmento gestionado de `opencode.json` (json-merge).
- `opencode/AGENTS.md`: bloque gestionado para `.opencode/AGENTS.md`.
- `bin/skills-hub.js`: CLI instalable del repo.
- `bin/copilot-credits.js`: CLI de análisis de AI Credits de GitHub Copilot.
- `scripts/sync.sh`: instalador por **copia** (rsync) + config gestionada de OpenCode.
- `scripts/install-opencode-config.mjs`: instala la config gestionada de OpenCode (merge).
- `scripts/check.sh`: detecta drift entre el repo y las copias instaladas.
- `scripts/import-copilot-skills.sh`: bootstrap para importar skills locales existentes al repo.

## Uso rápido

```bash
pnpm install
pnpm skills-hub status            # apps detectadas + auditoría del catálogo
pnpm skills-hub install --dry-run # plan de copia sin tocar disco
pnpm skills-hub install           # copia las skills a las apps locales
pnpm skills-hub registry list     # indice de skills para delegadores/agentes
pnpm skills-hub check             # ¿hay drift entre repo y copias?
```

Equivalentes directos:

```bash
./scripts/sync.sh --dry-run
./scripts/sync.sh
./scripts/sync.sh --app=claude
./scripts/check.sh
./scripts/doctor.sh
./scripts/doctor-skills.sh
./scripts/lint.sh
./scripts/import-copilot-skills.sh --dry-run
```

## Instalación en una máquina nueva

```bash
git clone <repo> ~/sources/skills-hub   # en disco LOCAL, nunca en un NAS
cd ~/sources/skills-hub
pnpm install
pnpm skills-hub install
```

`sync.sh` detecta qué apps existen (`~/.copilot`, `~/.claude`, `~/.agents`, OpenCode) y copia a cada una las fuentes que declara `config/apps.json`. Las apps no detectadas se omiten (usa `--include-missing` para forzar).

## CLI de instalación

La CLI oficial del repo es `skills-hub`.

```bash
pnpm skills-hub install [--app=<id>] [--dry-run] [--include-missing] [--verbose]
pnpm skills-hub sync    # alias de install
pnpm skills-hub status  # detección de apps + auditoría del catálogo
pnpm skills-hub doctor
pnpm skills-hub doctor-skills
pnpm skills-hub registry list [--json]
pnpm skills-hub registry refresh [--output=<path>]
pnpm skills-hub registry check [--json]
pnpm skills-hub lint
pnpm skills-hub check
```

Flags de `install`/`sync`:

- `--app=<id>` limita a una app concreta
- `--dry-run` muestra el plan sin tocar disco
- `--include-missing` instala aunque la app no se detecte
- `--verbose` detalla cada operación de rsync

## Skill registry

Inspirado por el modelo index-first de DeerFlow, `skills-hub` puede generar un
índice local de skills sin compactar sus reglas:

```bash
pnpm skills-hub registry list
pnpm skills-hub registry list --json
pnpm skills-hub registry refresh
pnpm skills-hub registry check
```

`registry refresh` escribe `.skills-hub/skill-registry.md` por defecto. Ese
archivo es un índice para delegadores: incluye nombre, descripción, scope, apps
expuestas, herramientas declaradas, coste aproximado en tokens y la ruta exacta
de `SKILL.md`. La skill completa sigue siendo la fuente de verdad y debe leerse
antes de ejecutar el trabajo.

`registry check` valida el catálogo para el discovery progresivo (el modelo que
usan VS Code Copilot Agent Skills y DeerFlow): nombre en minúsculas/dígitos/
guiones y ≤64 caracteres, description obligatoria y ≤1024 caracteres (los
límites de Copilot; si se superan la skill se ignora en silencio), description
con triggers útiles (≥40 caracteres) y cuerpo ≤300 líneas. Todo se reporta como
warnings sin bloquear.

La exposición por app es la efectiva tras la instalación. Cada app declara en
`config/apps.json` los proyectos y variantes que consume; VS Code/Copilot usa
solo `copilot-only`, mientras Claude y Agents mantienen sus variantes propias.
El orden de proyectos y variantes se resuelve con `config/projects.json`
(mismo criterio que la copia por rsync). El registry
lista esos casos en la sección `Overrides` y `registry check` los reporta como
warnings sin bloquear; `doctor-skills` ejecuta `registry check` automáticamente.

Notas:

- respeta `config/apps.json` como fuente de exposición por plataforma
- usa `pnpm skills-hub ...` dentro del repo; `pnpm exec` no expone el binario del paquete raíz

## Análisis de créditos de Copilot

`copilot-credits` es una CLI local que lee los archivos de sesión de GitHub Copilot y calcula el consumo de **AI Credits** aplicando la tabla de precios oficial de GitHub. No sube ningún dato; todo el procesamiento es local.

```bash
copilot-credits              # últimos 30 días
copilot-credits --days=90
copilot-credits --model=claude-sonnet-4.6
copilot-credits --sessions   # desglose por sesión
copilot-credits --json       # salida JSON
```

Salida de ejemplo:

```
GitHub Copilot AI Credits Usage
Period : 2026-03-15 → 2026-06-07 (last 90 days)
Sessions: 71  |  Plan monthly allotment: Pro=1,500  Pro+=7,000  Max=20,000

MODEL                    REQUESTS    INPUT TOKENS    CACHE READ   CACHE WRITE  OUTPUT TOKENS    CREDITS       USD
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────
claude-sonnet-4.5             744      48,834,450    43,778,114     2,585,949        447,037    17604.0   $176.04
gpt-5.4                       190      18,304,795    16,253,184             0        105,399     5140.6    $51.41
claude-haiku-4.5              576      33,611,939    31,858,398             0        178,865     3769.2    $37.69
claude-sonnet-4.6             262      10,974,559    10,221,750             0         74,362     3710.6    $37.11
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────
TOTAL                       1,773     111,728,535   102,111,446     2,585,949        805,670    30224.9   $302.25
```

**Fuente de datos:** `~/.copilot/session-state/<id>/events.jsonl` — el campo `session.shutdown` que Copilot escribe al cerrar cada sesión contiene el desglose de tokens por modelo.

**Precios:** tabla oficial de GitHub (`docs.github.com/en/copilot/reference/copilot-billing/models-and-pricing`). Los modelos sin entrada en la tabla se marcan con `*` y usan un precio estimado.

**1 AI Credit = $0.01 USD.** El allotment mensual incluido depende del plan: Pro 1.500 · Pro+ 7.000 · Max 20.000 · Business 1.900 · Enterprise 3.900 (cifras a junio 2026).

## Flujo profesional recomendado

Lee `CONTRIBUTING.md` y valida cambios locales antes de abrir PR:

```bash
./scripts/doctor-skills.sh
./scripts/doctor.sh
./scripts/lint.sh
./scripts/check.sh
```

## Calidad automatizada

- CI ejecuta `./scripts/lint.sh` en cada push y pull request.
- El objetivo de CI es validar scripts y convenciones del repo sin depender de destinos locales.
- `./scripts/validate-skills.sh` valida reglas semánticas básicas: límite de 300 líneas por `SKILL.md`, naming no obsoleto y referencias legacy controladas.
- `./scripts/doctor-skills.sh` audita el catálogo canónico: alineación carpeta/frontmatter, fuentes expuestas por app y colisiones de nombre por plataforma.

## Fuente canónica y modelo de exposición

Este repo es la **fuente canónica** de authoring:

- `projects/casa/` y `projects/workflow/` contienen las skills reales
- `config/projects.json` define los límites y fuentes de cada proyecto
- `config/apps.json` define qué proyectos y variantes expone cada app
- `config/sync-map.sh` mantiene solo contenido legacy copiable

Las rutas locales de apps (`~/.copilot/skills`, `~/.claude/skills`, `~/.agents/skills`, OpenCode) son **targets de exposición** (copias), no sitios de mantenimiento manual. Editar la copia instalada se considera drift y `check.sh` lo detecta.

Reglas:

- un nombre de skill canónico por directorio
- carpeta y frontmatter `name` deben coincidir
- no duplicar el mismo nombre de skill dentro del conjunto expuesto a una misma app
- tras renames, actualizar también prompts, config gestionada y referencias

## Criterio de clasificación

- Una skill del intranet va a `projects/casa/skills/common`; una skill portable va a `projects/workflow/skills/common`.
- Solo va a la variante `copilot-only` o `claude-only` del proyecto correspondiente si **depende** de esa plataforma concreta.
- No mezclar configuración de máquina dentro de `projects/*/skills/`; eso queda en el tooling o fuera del repo.

## Convención global de package manager

Para proyectos JavaScript/TypeScript, la convención por defecto del catálogo es:

- usar `pnpm` como package manager recomendado
- evitar ejemplos nuevos con `npm` salvo que la skill documente compatibilidad heredada o tooling externo
- reflejar la política de seguridad de dependencias cuando una skill cubra bootstrap o setup de proyecto

Política recomendada:

```yaml
minimumReleaseAge: 10080
minimumReleaseAgeStrict: true
minimumReleaseAgeIgnoreMissingTime: false
```

Excepciones:

- si un repo ya declara otro package manager, la skill debe respetar el repo y explicitarlo
- ejemplos con `npm` solo son válidos si el contexto depende de tooling o fixtures externos que aún lo requieran

## Contenido legacy copiable

`prompts/` se trata como contenido copiable y se sincroniza con `rsync` (par definido en `config/sync-map.sh`). `sync.sh` ejecuta ese paso después de copiar las skills.

## GitFlow del repositorio

- `main` para releases
- `develop` para integración
- `feature/*` desde `develop`
- `release/vX.Y.Z` desde `develop`
- `hotfix/*` desde `main`

Referencia completa: `GITFLOW.md`, `BRANCH_PROTECTION.md`, `RELEASING.md`.

## Historial de cambios

- Ver `CHANGELOG.md` para cambios notables del proyecto.
- Ver `RELEASING.md` para el proceso de versionado y publicación.

## Gobernanza y seguridad

- Ver `CONTRIBUTING.md` para flujo de contribución.
- Ver `SECURITY.md` para reporte responsable de vulnerabilidades.
- Ver `.github/CODEOWNERS` para propiedad de revisión por rutas.
- Ver `BRANCH_PROTECTION.md` para enforcement de checks en `main`.
