# Comandos CLI - Trello API Client

Cliente CLI del clon de Trello de la intranet `.CASA`. El token se guarda en `~/.trello/`.

## Autenticación

```bash
# Login interactivo (guarda token en ~/.trello/token)
trello login

# Login con credenciales directas
trello login --email admin@trello.casa --password xxxx --mfa-code 123456

# Logout (elimina token)
trello logout

# Ver info de la sesión actual
trello whoami
```

## Boards

```bash
# Listar todos los boards
trello board list

# Crear nuevo board
trello board create "SDD Feature-X" --desc "Tablero para la feature X" --color "#0052CC"

# Obtener board por ID
trello board get <board-id>

# Actualizar board
trello board update <board-id> --title "Nuevo título" --desc "Nueva descripción"

# Archivar board (soft delete)
trello board archive <board-id>

# Eliminar board (hard delete con cascada)
trello board delete <board-id>
```

## Lists (Columnas)

```bash
# Crear lista en un board
trello list create <board-id> "In Progress"

# Listar listas de un board
trello list list <board-id>

# Reordenar listas
trello list reorder <board-id> --ids "list-1,list-2,list-3"

# Actualizar lista
trello list update <list-id> --title "Nueva columna"

# Eliminar lista (cascada: elimina cards)
trello list delete <list-id>
```

## Cards

```bash
# Crear card en lista
trello card create <list-id> "Implementar login" --desc "Descripción de la tarea"

# Atributos opcionales
trello card create <list-id> "Task title" \
  --desc "Descripción" \
  --labels "Backend,API" \
  --due-date "2026-06-01"

# Mover card a otra lista (por ID + posición)
trello card move <card-id> --list <new-list-id> --position 0

# Mover card entre columnas (workflow por nombre)
trello card move-to <card-id> --column "In Progress"

# Actualizar card
trello card update <card-id> \
  --title "Nuevo título" \
  --desc "Nueva descripción" \
  --due-date "2026-07-01"

# Obtener card por ID
trello card get <card-id>

# Listar cards de una lista
trello card list <list-id>

# Eliminar card
trello card delete <card-id>
```

## Custom Fields

```bash
# Crear campo select con opciones, visible en la card
trello field create <board-id> "Estado" --type select \
  --options "Pendiente,En Progreso,Hecho" \
  --show-on-card

# Otros tipos: text, number, checkbox, date
trello field create <board-id> "Prioridad" --type text
trello field create <board-id> "Horas" --type number
trello field create <board-id> "Aprobado" --type checkbox
trello field create <board-id> "Fecha" --type date

# Establecer valor en card
trello field set <card-id> <field-id> "En Progreso"

# Obtener campos de un board
trello field list <board-id>
```

## Integraciones

```bash
# --- SDD ---
trello sdd-init <feature-name> --columns "Propuesta,Design,Apply,Review,Done"
trello sdd-tasks <sdd-tasks-file.json>
trello sdd-sync --feature <feature-name>

# --- Atlas ---
trello atlas-document <board-id>
trello atlas-sync --board <board-id>

# --- Portal ---
trello portal-update-badge <board-id> --status "active" --count 12
trello portal-snapshot --boards 5 \
  --output /mnt/nas/webs/portal.casa/data/snapshot.json
```

Ver [`sdd-integration.md`](sdd-integration.md) para el mapeo completo de fases SDD ↔ columnas.

## Configuración del cliente

Archivo `~/.trello/config.json`:

```json
{
  "baseUrl": "http://localhost:3002",
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "uuid-1234",
    "email": "user@trello.casa",
    "name": "Full Name"
  }
}
```
