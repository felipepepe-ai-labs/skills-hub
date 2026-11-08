---
name: deploy-casa
description: >
  Deploys a web application to the .casa intranet: compiles the frontend, copies
  artifacts to /mnt/nas/webs/<domain>/, configures nginx on maya, adds DNS on
  both Pi-holes, and registers the app in portal.casa.
  Trigger: user says "deploy", "install on nginx", "add to portal.casa",
  "publish on the intranet", or similar.
license: Apache-2.0
metadata:
  author: Felipe Perez
  version: "1.1"
---

## When to use this skill

- The user wants to deploy a web app on the `.casa` local network
- The user says "install on nginx", "add to portal.casa", "publish on the intranet"
- A new frontend was just developed and needs to be served on the network

---

## Reference infrastructure

| Machine | IP | Role |
|---------|----|------|
| maya | 192.168.1.55 | nginx server, NAS mounted at `/mnt/nas` |
| pihole1 | 192.168.1.53 | Primary DNS |
| pihole2 | 192.168.1.54 | Secondary DNS, portal.casa |

**Key paths on maya:**

| Path | Purpose |
|------|---------|
| `/mnt/nas/webs/<domain>/` | Production root for static frontends |
| `/mnt/nas/sources/<project>/` | Source code |
| `/etc/nginx/sites-available/` | nginx configs |
| `/etc/dnsmasq.d/local.conf` | DNS on each Pi-hole (via SSH) |
| `/etc/hosts` | Local DNS workaround on maya |
| `/mnt/nas/webs/portal.casa/index.html` | portal.casa dashboard |

**Occupied ports on maya:**

| Port | Service |
|------|---------|
| 3000 | obsidian-api |
| 3001 | poc-trello backend |
| 9100 | node_exporter |
| 5432 | postgresql |

---

## Deployment protocol

### Step 1 — Gather information

Before starting, determine:
- **Domain**: `<name>.casa` (follow existing pattern: `trello.casa`, `oficina.casa`…)
- **Type**: static frontend only / frontend + Node.js backend
- **Frontend framework**: Angular (`pnpm exec ng build`), React/Vite (`pnpm run build`), other
- **Backend port** (if applicable): choose a free port on maya

### Step 2 — Build frontend

```bash
cd /mnt/nas/sources/<project>/frontend   # or root if direct SPA

# Install dependencies if node_modules doesn't exist
pnpm install

# Angular
pnpm exec ng build --configuration production

# Vite / React
pnpm run build
```

**Typical output directory:**
- Angular: `dist/<name>/browser/`
- Vite: `dist/`

### Step 3 — Copy to production on NAS

```bash
mkdir -p /mnt/nas/webs/<domain>.casa
cp -r <dist-path>/. /mnt/nas/webs/<domain>.casa/
```

### Step 4 — Configure nginx on maya

Create `/etc/nginx/sites-available/<domain>.casa`:

```nginx
server {
    listen 80;
    server_name <domain>.casa;

    root /mnt/nas/webs/<domain>.casa;
    index index.html;

    # Only if there is a Node.js backend:
    location /api/ {
        proxy_pass http://127.0.0.1:<backend-port>;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

```bash
sudo ln -sf /etc/nginx/sites-available/<domain>.casa /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### Step 5 — Add DNS on both Pi-holes

```bash
# On pihole1 (192.168.1.53) and pihole2 (192.168.1.54):
for host in 192.168.1.53 192.168.1.54; do
  ssh $host "echo 'address=/<domain>.casa/192.168.1.55' | sudo tee -a /etc/dnsmasq.d/local.conf && sudo systemctl restart pihole-FTL && echo ok"
done
```

> ⚠️ **DO NOT use `/etc/pihole/custom.list`** — it does not work in this installation.
> The actual DNS is in `/etc/dnsmasq.d/local.conf` on each Pi-hole.

### Step 6 — Add entry to /etc/hosts on maya

```bash
echo "192.168.1.55 <domain>.casa" | sudo tee -a /etc/hosts
```

> ⚠️ **Required**: maya has `mdns4_minimal [NOTFOUND=return]` in `/etc/nsswitch.conf`
> which prevents resolving `.casa` domains via DNS. Without this entry, maya itself cannot resolve the domain.

### Step 7 — Node.js backend (if applicable)

Create systemd service at `/etc/systemd/system/<project>.service`:

```ini
[Unit]
Description=<project> backend
After=network.target postgresql.service

[Service]
Type=simple
User=felipe
WorkingDirectory=/mnt/nas/sources/<project>/backend
EnvironmentFile=/mnt/nas/sources/<project>/backend/.env
ExecStart=/home/felipe/.nvm/versions/node/v22.22.0/bin/node dist/src/server.js
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> ⚠️ The compiled entry point is `dist/src/server.js`, **not** `dist/server.js`
> (tsc preserves the `src/` directory structure).

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now <project>
sudo systemctl status <project>
```

**After each build, copy static assets manually** (tsc doesn't include them):
```bash
# Example: openapi.yaml
cp -r src/openapi dist/src/openapi
```

### Step 8 — Add card to portal.casa

Edit `/mnt/nas/webs/portal.casa/index.html`:

1. Increment `<span class="section-badge">N</span>` in the "Services" section
2. Add card before the closing `</div></div>` of the section:

```html
<a class="card" href="http://<domain>.casa" data-check="http://<domain>.casa" data-name="<search keywords>">
  <div class="card-top"><div class="card-icon">🎯</div><div class="dot checking"></div></div>
  <div class="card-name"><Visible Name></div>
  <div class="card-desc"><Brief description.></div>
  <div class="card-meta"><span class="card-url"><domain>.casa</span><span class="badge">maya</span></div>
</a>
```

### Step 9 — Final verification

```bash
# Frontend accessible
curl -s -H "Host: <domain>.casa" http://192.168.1.55/ | head -3

# API (if applicable)
curl -s http://localhost:<backend-port>/health

# DNS from another device on the network
# (from maya use the IP directly because nsswitch.conf interferes)
```

---

## Quick checklists

### Frontend-only deploy
- [ ] `pnpm install` + build
- [ ] Copy dist to `/mnt/nas/webs/<domain>.casa/`
- [ ] nginx site + reload
- [ ] DNS on pihole1 and pihole2 (`dnsmasq.d/local.conf` + restart pihole-FTL)
- [ ] `/etc/hosts` on maya
- [ ] Card in portal.casa

### Frontend + Node.js backend deploy
- Everything above, plus:
- [ ] `.env` with a free port (see port table)
- [ ] `pnpm run build` on backend
- [ ] Copy static assets to dist (`openapi.yaml`, etc.)
- [ ] Systemd service with entry point `dist/src/server.js`
- [ ] `sudo systemctl enable --now <project>`
- [ ] Proxy `/api/` in nginx pointing to the backend port

### Security (all deploys)
- [ ] No real secrets in `.env` or in the repo (use Infisical)
- [ ] Service runs as a non-root user
- [ ] Backend reachable only through the nginx proxy, no extra open ports
- [ ] Least-privilege permissions on deployed files and configs

---

## Updating an existing deployment

```bash
# 1. Rebuild frontend
cd /mnt/nas/sources/<project>/frontend && pnpm exec ng build --configuration production

# 2. Sync to production
cp -r dist/<name>/browser/. /mnt/nas/webs/<domain>.casa/

# 3. If there is a backend: rebuild and restart
cd /mnt/nas/sources/<project>/backend && pnpm run build
cp -r src/openapi dist/src/openapi   # static assets
sudo systemctl restart <project>
```

No need to touch nginx, DNS, or portal.casa on updates.
