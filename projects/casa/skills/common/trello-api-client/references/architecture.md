# Arquitectura del Servicio Trello Clone

## Estado Actual ✅

**Fecha:** 2026-05-27 | **Puerto:** 3002 | **Estado:** OPERATIVO 🟢

## Esquema de Infraestructura

```
┌─────────────────────────────────────────────────────────┐
│                    maya (192.168.1.55)                  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Trello Clone - Servicio NATIVO (systemd)       │   │
│  │                                                 │   │
│  │  systemctl poc-trello                           │   │
│  │  node dist/src/server.js (puerto 3002)          │   │
│  │  User: felipe                                   │   │
│  │  ─────────────────────────────────────────────  │   │
│  │  Código: /mnt/nas/sources/pocs/poc-trello/      │   │
│  │  Frontend: /mnt/nas/webs/trello.casa/ (nginx)   │   │
│  │                                                 │   │
│  │  DB: PostgreSQL nativa (5432)                   │   │
│  │  User: trello | DB: poc_trello                  │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Nginx (Proxy & Static)                         │   │
│  │                                                 │   │
│  │  trello.casa → puerto 80                        │   │
│  │  / → frontend estático                          │   │
│  │  /api/ → proxy → localhost:3002                 │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## Componentes

### 1. Backend (Node + Express)

| Dato | Valor |
|------|-----:--|
| **Framework** | Express 4.x |
| **Runtime** | Node.js v22.22.0 |
| **Lenguaje** | TypeScript compilado |
| **Compilación** | tsc → dist/src/ |
| **Puerto** | 3002 |
| **User** | felipe |
| **Type** | **systemd nativo** (NO Docker) |

### 2. Service Systemd

```ini
# /etc/systemd/system/poc-trello.service
[Unit]
Description=poc-trello backend
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=felipe
WorkingDirectory=/mnt/nas/sources/poc-trello/backend
EnvironmentFile=/mnt/nas/sources/poc-trello/backend/.env
ExecStart=/home/felipe/.nvm/versions/node/v22.22.0/bin/node dist/src/server.js
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 3. Nginx Proxy (maya)

```nginx
# /etc/nginx/sites-available/trello.casa
server {
    listen 80;
    server_name trello.casa;

    root /mnt/nas/webs/trello.casa;
    index index.html;

    location /api/ {
        proxy_pass http://127.0.0.1:3002;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api-docs {
        proxy_pass http://127.0.0.1:3002;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### 4. Base de Datos (PostgreSQL nativa)

| Dato | Valor |
|------|-----:--|
| **Engine** | PostgreSQL nativa |
| **Puerto** | 5432 |
| **DB** | poc_trello |
| **User** | trello |
| **Socket** | /var/run/postgresql/.s.PGSQL.5432 |
| **Tabla Owner** | trello |

**Tablas:**
```sql
-- Lista completa
sudo -u postgres psql -d poc_trello -c '\dt'

# results:
# - users
# - auth_sessions
# - boards
# - lists
# - cards
# - custom_fields
# - card_custom_field_values
```

## Configuración

### Environment Variables

```env
# Ubicación: /mnt/nas/sources/pocs/poc-trello/backend/.env

# Database
DATABASE_URL=postgresql://trello:trello2026@localhost:5432/poc_trello

# Server
PORT=3002
CORS_ORIGIN=http://trello.casa

# Auth
JWT_SECRET=<random-secret>
JWT_EXPIRY=24h
ADMIN_EMAIL=admin@trello.casa
ADMIN_PASSWORD=<secure-password>

# MFA (opcional)
MFA_SECRET=<mfa-secret>
```

### Integraciones Externas

| Servicio | URL | Uso |
|---------|-----|-----|
| **Infisical** | http://infisical.casa | Gestión de secretos |
| **Keycloak** | http://auth.casa | Auth externo (futuro) |

## Swagger / API Docs

| Item | Valor |
|------|-----:--|
| **URL** | http://localhost:3002/api-docs |
| **Spec YAML** | `/mnt/nas/sources/pocs/poc-trello/backend/src/openapi/openapi.yaml` |
| **Tamaño** | ~18KB |
| **Status** | ✅ Activo y accesible |

**Probar Swagger:**
```bash
# Local
curl http://localhost:3002/health

# Via nginx
curl http://trello.casa/api/boards

# Docs
open http://trello.casa/api-docs
# o
open http://localhost:3002/api-docs
```

## Logs y Diagnóstico

### Ver Estado
```bash
sudo systemctl status poc-trello
```

### Ver Logs (tiempo real)
```bash
sudo journalctl -u poc-trello -f
```

### Ver Logs (últimos 50)
```bash
sudo journalctl -u poc-trello -n 50 --no-pager
```

### Ver Puerto
```bash
sudo ss -tlnp | grep 3002
```

### Reiniciar Servicio
```bash
sudo systemctl restart poc-trello
```

### Recargar Nginx
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Database Health
```bash
sudo -u postgres psql -d poc_trello -c "SELECT 1"
```

## Troubleshooting

| Problema | Causa | Solución |
|---|--|:--|
| **404 Not Found** | Servicio caído | `sudo systemctl restart poc-trello` |
| **EADDRINUSE** | Puerto ocupado | Verificar: `ss -tlnp \| grep 3002` |
| **DB Connection** | PostgreSQL caído | `sudo systemctl status postgresql` |
| **Nginx error** | Config inválida | `sudo nginx -t` |
| **Permission denied** | Archivos de root | `sudo chown -R felipe:felipe /mnt/nas/sources/pocs/poc-trello` |

## Comparación: Antes vs Ahora

| Antes ❌ | Ahora ✅ |
|---|--:----|
| Docker container | Servicio systemd nativo |
| Puerto 3001 | Puerto 3002 |
| .env falló | .env correcto |
| Service fallando | Service 200 OK |

## Referencias

- **Código:** `/mnt/nas/sources/pocs/poc-trello/`
- **Backend:** `/mnt/nas/sources/pocs/poc-trello/backend/`
- **Frontend:** `/mnt/nas/webs/trello.casa/`
- **Service:** `/etc/systemd/system/poc-trello.service`
- **Nginx:** `/etc/nginx/sites-available/trello.casa`
- **OpenAPI:** `/mnt/nas/sources/pocs/poc-trello/backend/src/openapi/openapi.yaml`

