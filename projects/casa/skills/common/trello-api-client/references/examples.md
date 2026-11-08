# Ejemplos de Uso - Trello API Client

## Flujo SDD Completo

```bash
# 1. Inicializar feature
sdd new "auto-deploy-pipeline"

# 2. Sincronizar con Trello
trello sdd-init "auto-deploy-pipeline" \
  --columns "Propuesta,Design,Apply,Review,Done"

# 3. Crear tasks del SDD
trello sdd-tasks sdd-tasks.json

# 4. Desarrollar → mover cards
trello card move-to <card-id> --column "In Progress"

# 5. Verificar → mover a review
trello card move-to <card-id> --column "Review"

# 6. Archivar → documentar en Atlas
sdd archive "auto-deploy-pipeline"
trello board archive <board-id>
trello atlas-document <board-id>
```

## Flujo Manual Simple

```bash
# Crear board
trello board create "Migración a NextJS" --desc "Actualizar frontend"

# Crear columnas
trello list create <board-id> "Backlog"
trello list create <board-id> "In Progress"
trello list create <board-id> "Done"

# Crear cards
trello card create <list-id> "Instalar NextJS" --desc "v14 con App Router"
trello card create <list-id> "Migrar componentes" --desc "Pages → App Router"

# Mover tasks
trello card move-to <card-id> --column "In Progress"
trello card move-to <card-id> --column "Done"
```
