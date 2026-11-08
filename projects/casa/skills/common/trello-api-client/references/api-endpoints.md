# API Trello Clone - Referencia de Endpoints

Base: `http://localhost:3002` (proxy nginx: `http://trello.casa/api`). Auth por `Authorization: Bearer <token>`.

## Autenticación

```http
POST /api/auth/login
Body: { email: string, password: string, mfa?: string }
Response: { accessToken: string, refreshToken: string, user: User }

POST /api/auth/refresh
Body: { refreshToken: string }
Response: { accessToken: string, refreshToken: string }

POST /api/auth/logout          → 204
GET  /api/auth/me              → { user: { id, email, name, mfaEnabled } }
GET  /api/auth/mfa/setup       → { qrCodeDataUrl: string, secret: string }
POST /api/auth/mfa/verify      Body: { tempToken, code } → { accessToken, refreshToken }
POST /api/auth/mfa/reconfigure
```

## Boards

```http
GET    /api/boards                 → Board[]
POST   /api/boards                 Body: { title, description?, background? } → Board (201)
GET    /api/boards/:id             → Board
PUT    /api/boards/:id             Body: { title?, description?, background? } → Board
DELETE /api/boards/:id             → 204
```

## Lists

```http
GET    /api/boards/:boardId/lists           → BoardList[]
POST   /api/boards/:boardId/lists           Body: { title } → BoardList (201)
PATCH  /api/boards/:boardId/lists/reorder   Body: { orderedIds: string[] } → BoardList[]
PUT    /api/lists/:id                        Body: { title?, position? } → BoardList
DELETE /api/lists/:id                        → 204
GET    /api/lists                            → BoardList[] (standalone)
```

## Cards

```http
GET    /api/lists/:listId/cards             → Card[]
POST   /api/lists/:listId/cards             Body: { title, description?, dueDate?, labels? } → Card (201)
PATCH  /api/lists/:listId/cards/reorder     Body: { orderedIds: string[] } → Card[]
PATCH  /api/cards/:id/move                   Body: { listId, position } → Card
GET    /api/cards/:id                        → Card
PUT    /api/cards/:id                        Body: { title?, description?, dueDate?, labels? } → Card
DELETE /api/cards/:id                        → 204
POST   /api/cards                            Body: { listId, title, ... } → Card (201, standalone)
```

## Custom Fields

```http
GET    /api/boards/:boardId/custom-fields   → CustomField[]
POST   /api/boards/:boardId/custom-fields   Body: { name, type, options? } → CustomField (201)
PUT    /api/custom-fields/:id                Body: { name?, options?, position?, showOnCard? } → CustomField
DELETE /api/custom-fields/:id                → 204
```

`type`: `'text' | 'number' | 'checkbox' | 'date' | 'select'`

## Card Field Values

```http
GET    /api/cards/:cardId/field-values              → CardFieldValue[]
POST   /api/cards/:cardId/field-values              Body: { fieldId, value } → CardFieldValue
PUT    /api/cards/:cardId/field-values/:fieldId     Body: { value } → CardFieldValue
DELETE /api/cards/:cardId/field-values/:fieldId     → 204
```

## Health & Docs

```http
GET /health      → estado del servicio
GET /api-docs    → Swagger UI
```

## Códigos de Error

| Código | Causa | Solución |
|--------|-------|----------|
| 401 Unauthorized | Token expirado o inválido | `trello login` |
| 403 Forbidden | Sin permiso sobre el board | Verificar ownership o contactar admin |
| 404 Not Found | ID inexistente | Verificar ID de board/list/card |
| 400 Bad Request | Datos inválidos | Revisar formato de campos requeridos |
| 500 Internal | Error del servidor | `sudo journalctl -u poc-trello` |
