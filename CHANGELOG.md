# Changelog

Todos los cambios notables de este proyecto se documentaran en este archivo.

El formato esta basado en [Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/),
y este proyecto adhiere a [Semantic Versioning](https://semver.org/lang/es/).

## [Sin publicar]

### Añadido

- Se documentan 19 skills de terceros (bundle HyperFrames y GSAP oficial) como dependencias externas en `config/external-skills.json` y `docs/external-skills.md`, sin copiar su contenido al hub.

## [5.0.0] - 2026-08-22

### Añadido

- Soporte para el CLI real de OpenSpec (`@fission-ai/openspec` ≥1.6): `sdd-init` ahora ejecuta `openspec init --tools all` y bifurca el schema por defecto para añadir un artefacto `verify` propio; `sdd new`/`apply`/`archive` delegan en `/opsx:propose`/`apply`/`archive` en vez de sub-agentes propios.
- `sdd-verify` gana una segunda opinión obligatoria (`code-reviewer`, `judgment-day`, `security-review`, `silent-failure-hunter`) en un modelo LLM distinto al usado en `sdd apply`, más e2e con Playwright y captura de pantalla por test.
- Engram vuelve como memoria de empresa aditiva (no como modo de persistencia): `sdd`, `sdd-init` y `sdd-verify` guardan una observación en cada transición de fase, sin sustituir a `openspec/` ni bloquear el ciclo si Engram no está disponible.
- `enrich-us`: modo Service Now (incidencias/peticiones/evolutivos por número de ticket vía MCP, con escritura de vuelta opcional) y modo Proyecto (entrevista al usuario cuando no hay ticket).
- `sdd-init` bootstrapea el CLI con `--tools all`, y `copilot` gana la variante `common` en `config/apps.json` — las skills ahora llegan a Copilot/VS Code, no solo a Claude.
- Detección e instalación de skills en OpenAI Codex CLI (`~/.codex/skills`).
- `check.sh` muestra la versión del frontmatter (instalada vs. repo) en cada drift detectado.
- `session-start` (claude-only) lee el grafo de código (`get_architecture`/`search_graph`) al principio de verdad, no solo como chequeo de frescura al final.
- Ciclo SDD completo verificado end-to-end contra el CLI real (init → propose → apply con TDD real → verify → archive).

### Cambiado

- `.gitignore` ahora ignora `.atl/`.
- TDD estricto (red→green→refactor) pasa a ser siempre obligatorio en el ciclo SDD, sin flag de configuración que lo desactive.
- Modo de persistencia único `openspec` — se retiran los modos `engram`/`hybrid`/`none` como backends alternativos de artefactos.
- Renombrada la skill `gitflow-casa` a `gitflow`.

### Eliminado

- Retiradas las skills bespoke `sdd-propose`, `sdd-spec`, `sdd-design`, `sdd-tasks`, `sdd-apply`, `sdd-archive`, `sdd-explore` (y sus 7 agentes en `agents/common/`) — su función la cubre ahora el CLI real de OpenSpec. Solo quedan `sdd` (orquestador), `sdd-init`, `sdd-verify` y `sdd-onboard`.
- Eliminado el proyecto `projects/casa/` por completo (`casa-ops`, `atlas-docs`, `mcp-builder`, `openai-docs`, `poc-init`, `skills-catalog-maintainer`); `react-doctor` y `red-team-offensive` se reubican en `projects/workflow` por ser dependencias reales de `sdd-verify`.
- `_shared/engram-convention.md` y `_shared/persistence-contract.md`, obsoletos tras el modo único `openspec`.

### Corregido

- `sdd-init` repetía a mano el texto del "executor boundary" y el formato de "structured envelope" en vez de referenciar `_shared/sdd-phase-common.md`; la regla duplicada además contradecía el propio contrato de salida compacto de la skill.

## [4.0.0] - 2026-07-19

### Añadido

- `scripts/sync.sh`: poda de skills huérfanas — al sincronizar cada app, elimina del destino los directorios de skills que ya no existen en ninguna fuente (se preservan `_*`/`.*` como `_shared`). Antes, las skills borradas de la fuente persistían para siempre en los targets (~100 huérfanas acumuladas en `~/.copilot/skills`).
- 8 skills autoradas localmente portadas a `projects/workflow/skills/claude-only` para cumplir el invariante "los targets no son lugares de autoría": `chained-pr`, `codebase-memory`, `cognitive-doc-design`, `comment-writer`, `headroom`, `issue-creation`, `skill-registry`, `work-unit-commits`.

### Corregido

- `scripts/check.sh` comparaba cada variante fuente contra el destino sin aplicar la precedencia de overrides de `sync.sh` (claude-only pisa a common), marcando falso drift en `session-start`/`session-end`; ahora compara solo la fuente efectiva por skill.
- `atlas-docs` (variantes `common` y `copilot-only`) referenciaba la carpeta inexistente `Projects/` del vault Atlas; corregido a `Proyectos/` (convención real del vault y de `session-start`/`session-end` v1.5).
- `scripts/sync.sh` fallaba con `mkdir: cannot create directory ... CLAUDE.md` al procesar pares de `sync-map.sh` cuyo destino es un archivo: ahora crea el directorio padre (`dirname`) para orígenes de tipo archivo.
- Eliminados de `config/sync-map.sh` los pares de `CLAUDE.md` y `copilot-instructions.md`, duplicados de los `configFiles` de `apps.json` que ya instala `install-behavior.sh`.

### Eliminado

- Pack DDIA completo (15 skills `ddia-*`), pack genérico (`agents-*`, `analysis-*`, `core-*`, `dev-*`, `docs-*`, `quality-skill-quality-gate`, `security-skill-security-gate`), suite HyperFrames/animación, y skills muertas o redundantes (`casa-atlas`, `casa-vault`, `skill-installer`, `infisical-admin-mcp`, `sdd-proposal`, `branch-pr`, `nueva-casa`) de `common`/`claude-only`; las variantes `copilot-only` no se tocan.

### Cambiado

- Runbooks casa consolidados en una única skill router `casa-ops` (dominios, deploys docker/web, secretos Infisical, Trello, workflow) con carga bajo demanda desde `references/`.
- `gitflow-casa` absorbe el flujo de creación de PRs (`references/pr-workflow.md`).
- Descripciones de frontmatter recortadas a una línea (~250 caracteres máx.) manteniendo los triggers; ahorra ~7k tokens de contexto por sesión en Claude.
- Catálogo `copilot-only` alineado con la limpieza: 8 skills redundantes o muertas sustituidas por `casa-ops` + `atlas-docs`.
- `behavior/claude/CLAUDE.md` reconciliado con la versión compactada instalada (tablas de skills sustituidas por regla de triggers); añadido `output-styles/neutral.md` para que sync no lo borre.

## [3.0.0] - 2026-07-15

### Cambiado (breaking)

- Catálogo reestructurado de `skills/` plano a `projects/casa/skills/` + `projects/workflow/skills/`, con `config/projects.json` declarando proyectos + variantes en lugar de un array estático de fuentes.

### Añadido

- `projects/workflow/skills/claude-only/session-start`: modo onboarding (briefing "First contact" para proyectos sin memoria previa) y heurística de primer contacto. v1.4 → v1.5.
- `session-start`/`session-end` (`claude-only` y `common`): default "Atlas-first" — Atlas es la fuente primaria, Engram completa detalle temporal/táctico, Atlas gana en conflicto.
- `session-start`/`session-end`: integración con `codebase-memory-mcp` — chequeo de freshness del grafo de código y prompt de re-indexado (nunca automático).

### Eliminado

- Skill `core-grafos-memory` y `prompts/grafos-memory.prompt.md`.
- Integración CodeGraph (`codegraph init/sync/status/files/explore`) — reemplazada por `codebase-memory-mcp`.

### Corregido

- Ruta de páginas de proyecto en Atlas: `Projects/` → `Proyectos/` (la carpeta `Projects/` nunca existió en el vault; toda lectura/escritura de Atlas en sesión era un no-op silencioso).

## [2.8.0] - 2026-07-05

### Añadido

- `projects/workflow/skills/common/compliance-ops`: guardrail de cumplimiento normativo con entrevista guiada para HIPAA, SOC 2, GDPR y PCI-DSS; 13 archivos de referencia bajo `references/`.
- `agents/common/sdd-{explore,propose,spec,design,tasks,apply,verify,archive}`: 8 agentes de fase SDD versionados por primera vez con asignación de modelo (`opus`=design, `sonnet`=explore/propose/spec/apply/verify, `haiku`=tasks/archive) y Prompt Defense Baseline obligatorio.
- `behavior/claude/CLAUDE.md`: sección `### Compliance` en la tabla de auto-carga de skills (`compliance-ops`, `eu-gdpr`, `eu-ai-act`); reglas de solo inglés, ejecución silenciosa con resumen al final, y preguntas una a una antes de empezar.
- `behavior/copilot/copilot-instructions.md`: mismas reglas de comportamiento aplicadas (`## Language`, `## Pre-Task Protocol`, contrato de output actualizado).
- `config/sync-map.sh`: entradas para `behavior/claude/CLAUDE.md` y `behavior/copilot/copilot-instructions.md` como fuentes versionadas.
- `dependabot.yml`: `target-branch: develop` para respetar GitFlow.
- `actions/checkout`: bump v6 → v7 en `quality.yml` y `release.yml`.

### Cambiado

- `sdd-propose`: modelo opus → sonnet (tarea de escritura estructurada, no arquitectura).
- `sdd-tasks`: modelo sonnet → haiku (descomposición mecánica desde artefactos spec+design completos).

## [2.7.0] - 2026-07-04

### Añadido

- `skills-hub registry list|refresh|check [--json]`: registro index-first de skills inspirado en el modelo de carga progresiva de DeerFlow. `refresh` genera `.skills-hub/skill-registry.md` con nombre, descripción, scope, apps efectivas, coste aproximado en tokens y ruta exacta de cada `SKILL.md` (índice, no resumen de reglas).
- Exposición efectiva por app: cuando una fuente de plataforma (`claude-only`/`copilot-only`) contiene una skill homónima de `common`, gana la última fuente declarada en `config/apps.json` (semántica rsync). Los overrides se listan en la sección `Overrides` del índice.
- `registry check`: warnings de gobernanza y de límites de Copilot Agent Skills (name ≤64 en minúsculas/dígitos/guiones; description obligatoria ≤1024; description ≥40; cuerpo ≤300 líneas). Integrado en `doctor-skills.sh`.
- Contrato de tokens de salida en `behavior/copilot/copilot-instructions.md` y `behavior/claude/CLAUDE.md` (answer-first, sin eco de código, diffs en lugar de archivos completos).

### Arreglado

- `skills/common/remotion-to-hyperframes`: description reducida de 1240 a <1024 caracteres — superaba el límite de VS Code Copilot Agent Skills y la skill se ignoraba en silencio.

## [2.6.0] - 2026-06-28

### Añadido

- `projects/workflow/skills/claude-only/session-start`: bloque de intake al final del briefing — 4 preguntas (goal, constraints, approach, blockers). Claude se detiene y espera respuesta antes de comenzar cualquier tarea. v1.1.

### Arreglado

- `projects/workflow/skills/common/session-start`: reemplaza llamadas a `bash ~/.copilot/hooks/copilot/*.sh` con comandos git nativos — la skill ahora funciona en Claude, agents y Copilot sin dependencias de plataforma. Corrige campo `author` (era `gentleman-programming`). v1.3.

## [2.5.0] - 2026-06-26

### Añadido

- `projects/workflow/skills/common/redis-cache`: nueva skill portable para caching con Redis en Node.js/TypeScript. Cubre selección de estrategia (cache-aside/write-through/write-behind), diseño de claves, política de TTL, patrones de cliente node-redis v5 (RESP3, client-side caching), manejo de fallos y pitfalls comunes.

## [2.4.0] - 2026-06-14

### Añadido

- `DDIA_Skills/` y `token-efficient-skills-pack/` añadidos a `.gitignore`.

## [2.3.0] - 2026-06-14

### Añadido

- `projects/workflow/skills/common/critical-advisor`: skill de persona consejera con 7 reglas — cuestiona primero, etiqueta confianza, elimina frases de validación, mantiene posición bajo presión.
- `scripts/bootstrap.sh` + `scripts/patch-mcp.mjs`: instalación unificada para nueva máquina. Un solo comando (`pnpm bootstrap`) construye el MCP de grafos, sincroniza skills y parchea `~/.claude.json` y `~/.vscode/mcp/config.json` de forma idempotente.
- `pnpm bootstrap` registrado en `package.json`.
- Integración de grafos en `session-start` y `session-end` (claude-only + common): `grafos_recall` al inicio de sesión, `grafos_remember` al cierre. Ambos condicionales — no bloquean si grafos no está disponible. (Nota: esta integración fue eliminada en v3.0.0.)
- Pack DDIA en `projects/workflow/skills/common/ddia-*`: 15 skills compartidas para diseño y revisión de sistemas data-intensive, incluyendo router, modelos de datos, almacenamiento, replicación, sharding, transacciones, sistemas distribuidos, consenso, batch, streaming y ética de datos.

## [2.2.0] - 2026-06-07

### Añadido

- `bin/copilot-credits.js`: nueva CLI local para analizar el consumo de AI Credits de GitHub Copilot. Lee `~/.copilot/session-state/<id>/events.jsonl`, aplica la tabla de precios oficial de GitHub y muestra un resumen por modelo con tokens, créditos y coste en USD. Soporta `--days`, `--model`, `--sessions`, `--json`.
- Comando `copilot-credits` registrado en el campo `bin` de `package.json`.
- Sección `## Análisis de créditos de Copilot` en `README.md` con referencia de uso, ejemplo de salida y explicación de la fuente de datos.

## [2.1.0] - 2026-06-07

### Añadido

- Sub-capa de sub-agentes en `agents/` (`common` y `claude-only`): 4 agentes instalados en `~/.claude/agents/` — `planner`, `silent-failure-hunter`, `security-reviewer`, `build-error-resolver` (#21).
- `projects/workflow/skills/common/_shared/prompt-defense-baseline.md`: bloque canónico de defensa contra prompt injection, requerido en todas las definiciones de agente (#21).
- Soporte de `agentSources` y `agentInstallPath` en `config/apps.json` para la app `claude` (#21).
- Skills de cumplimiento normativo europeo: `eu-ai-act` y `eu-gdpr` en `projects/workflow/skills/common` (#19).
- Archivos de comportamiento versionados (`behavior/claude/CLAUDE.md`, `behavior/copilot/copilot-instructions.md`) con tabla de auto-carga de skills completa (#20).
- `scripts/install-behavior.sh`: instalador de archivos de comportamiento (copia segura sin `--delete`) integrado en el pipeline de sync (#20).
- `config/model-map.json`: configuración de scheduling de modelos según VRAM disponible.
- `skills/skills-manifest.json`: registro canónico de skills.
- `scripts/validate-skill-system.sh`: script de validación del sistema de skills en 7 fases.

### Cambiado

- 9 skills movidas de `copilot-only` a `common` (`atlas-docs`, `casa-*`, `infisical-*`, `project-workflow`, `trello-api-client`) — paridad de catálogo a 47+ skills por plataforma (#20).
- 24 skills reescritas con esquemas de salida fijos (campos delimitados) para reducir tokens de respuesta y eliminar variabilidad (#20).
- Todas las skills traducidas al inglés; regla de respuesta siempre en inglés aplicada en todos los skills y archivos de comportamiento (#20).
- `scripts/sync.sh` integra el ciclo de agentes con `rsync_file` sin `--delete` (preserva agentes de fuentes externas) (#21).
- `scripts/check.sh` y `scripts/doctor.sh` extendidos con sección de detección de drift de agentes (#21).
- Reducción de huella de tokens en skills de alta frecuencia: `skill-creator` −57%, `code-reviewer` −23%, `test-runner` −14%; límite máximo de 5 hallazgos en `code-reviewer` (#23).

### Corregido

- `doctor-skills.sh`: validación de nombres duplicados opera ahora por superficie de app, permitiendo variantes específicas de plataforma (ej. `session-start`) (#19).

## [2.0.0] - 2026-05-29

### Cambiado

- **Modelo de instalacion: de symlinks a copia.** Las skills ahora se **copian** (rsync) a cada app en `scripts/sync.sh`, en lugar de exponerse por symlink. Quedan en disco local de la maquina e independientes del clon. Tras editar una skill hay que re-ejecutar `sync` (BREAKING: ya no es instantaneo).
- Invariante de localidad: `scripts/lib/common.sh` incorpora `skills_hub_assert_local`, que aborta si el clon o algun destino vive en un filesystem de red (NFS/CIFS/SMB/sshfs). Aplicado en `sync.sh`, `check.sh` y `doctor.sh`.
- `check.sh` detecta drift comparando contenido repo<->copias (rsync -ani) y marca symlinks residuales como drift.
- `bin/skills-hub.js`: `install`/`sync` ejecutan `sync.sh`; `status` reutiliza la deteccion de `doctor.sh`.
- Reclasificacion del catalogo: el grupo SDD y las skills base de Claude se movieron de `copilot-only` a `common` (default = `common`).
- Modularizada la skill `trello-api-client` (SKILL.md de 616 a <100 lineas; detalle en `references/`).
- Author propio normalizado a `Felipe Pérez` en las skills propias.

### Aniadido

- Skills incorporadas al repo: `find-skills` y `mcp-builder` (common), `infisical-admin-mcp` (copilot-only).
- `scripts/install-opencode-config.mjs`: instala la config gestionada de OpenCode (json-merge + bloque markdown).
- Guardia anti-NAS `skills_hub_assert_local` en `scripts/lib/common.sh`.
- `scripts/doctor.sh` para diagnostico rapido de entorno local.
- Baseline de estandares de repositorio con `.editorconfig`, `.gitattributes` y `LICENSE`.
- Workflow de release por tags en `.github/workflows/release.yml`.
- Politica versionada de branch protection en `.github/branch-protection.main.json`.
- Script `scripts/setup-branch-protection.sh` para aplicar branch protection con GitHub CLI.
- Guia operativa en `BRANCH_PROTECTION.md`.

### Eliminado

- Motor de symlinks `scripts/link-skills.mjs` y `scripts/lib/link-skills-*.mjs`. La logica de config gestionada de OpenCode se traslado a `scripts/install-opencode-config.mjs`.
- Skill duplicada `mente-docs` (clon al 75% de `atlas-docs` con naming obsoleto `mente`; `atlas-docs` es la canonica).

## [1.1.0] - 2026-05-17

Release publicado en GitHub. El detalle de cambios no se registro en este
changelog en su momento; ver
<https://github.com/FelipePepe/skills-hub/releases/tag/v1.1.0>.

## [0.1.0] - 2026-04-13

### Aniadido

- Archivo de instrucciones globales para agentes en `.github/copilot-instructions.md`.
- Guia de contribucion en `CONTRIBUTING.md`.
- Plantilla de pull request en `.github/pull_request_template.md`.
- Plantillas de issue para bug y feature request en `.github/ISSUE_TEMPLATE/`.
- Workflow de calidad en GitHub Actions en `.github/workflows/quality.yml`.
- Script `scripts/lint.sh` para validar sintaxis Bash y formato de `SYNC_PAIRS`.

### Cambiado

- `README.md` ahora incluye flujo profesional recomendado y seccion de calidad automatizada.

[Sin publicar]: https://github.com/felipepepe-ai-labs/skills-hub/compare/v5.0.0...HEAD
[5.0.0]: https://github.com/felipepepe-ai-labs/skills-hub/compare/v4.0.0...v5.0.0
[4.0.0]: https://github.com/felipepepe-ai-labs/skills-hub/compare/v3.0.0...v4.0.0
[3.0.0]: https://github.com/FelipePepe/skills-hub/compare/v2.8.0...v3.0.0
[2.8.0]: https://github.com/FelipePepe/skills-hub/compare/v2.7.0...v2.8.0
[2.7.0]: https://github.com/FelipePepe/skills-hub/compare/v2.6.0...v2.7.0
[2.6.0]: https://github.com/FelipePepe/skills-hub/compare/v2.5.0...v2.6.0
[2.5.0]: https://github.com/FelipePepe/skills-hub/compare/v2.4.0...v2.5.0
[2.4.0]: https://github.com/FelipePepe/skills-hub/compare/v2.3.0...v2.4.0
[2.3.0]: https://github.com/FelipePepe/skills-hub/compare/v2.2.0...v2.3.0
[2.2.0]: https://github.com/FelipePepe/skills-hub/compare/v2.1.0...v2.2.0
[2.1.0]: https://github.com/FelipePepe/skills-hub/compare/v2.0.0...v2.1.0
[2.0.0]: https://github.com/FelipePepe/skills-hub/compare/v1.1.0...v2.0.0
[1.1.0]: https://github.com/FelipePepe/skills-hub/releases/tag/v1.1.0
[0.1.0]: https://github.com/FelipePepe/skills-hub/releases/tag/v0.1.0
