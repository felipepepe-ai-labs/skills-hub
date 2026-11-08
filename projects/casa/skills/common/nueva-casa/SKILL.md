---
name: nueva-casa
description: >
  Registers an existing service on the .casa intranet: adds DNS on both Pi-holes,
  creates the nginx config on maya (reverse proxy or static), and adds the card to
  portal.casa. No compilation — for services already running on a port or IP on the
  network. Trigger: "add X to the intranet", "create domain X.casa", "point X.casa
  to this port", "register service in portal".
license: Apache-2.0
metadata:
  author: Felipe Perez
  version: "1.0"
---

## When to Use This Skill

- A service is already running (Docker, Node, Python…) and needs to be accessed via `name.casa`
- You want to add a `.casa` alias for an existing IP/port
- `deploy-casa` is overkill because there is nothing to compile or deploy

---

## Required Information Before Starting

1. **Domain name**: `<name>.casa`
2. **Target IP**: which machine is it running on? (maya=192.168.1.55, pihole2=192.168.1.54…)
3. **Port**: if reverse proxy, which port does the service listen on?
4. **Nginx type**: reverse proxy to a port / serve existing static files
5. **Description and emoji** for the portal.casa card

---

## Reference Infrastructure

| Machine | IP | Nginx |
|---------|----|-------|
| maya | 192.168.1.55 | `/etc/nginx/sites-available/` |
| pihole2 | 192.168.1.54 | SSH + nginx |

**DNS**: `/etc/dnsmasq.d/local.conf` on pihole1 (192.168.1.53) and pihole2 (192.168.1.54)

---

## Step 1 — DNS on Both Pi-holes

```bash
for host in 192.168.1.53 192.168.1.54; do
  ssh $host "echo 'address=/<name>.casa/<target-IP>' \
    | sudo tee -a /etc/dnsmasq.d/local.conf \
    && sudo systemctl restart pihole-FTL \
    && echo ok"
done
```

> ⚠️ Always use `dnsmasq.d/local.conf`. Do NOT use `/etc/pihole/custom.list`.

---

## Step 2 — Nginx on the Target Machine

### Case A: Reverse Proxy (service on a local port)

In `/etc/nginx/sites-available/<name>.casa` (on maya or via SSH if it's on pihole2):

```nginx
server {
    listen 80;
    server_name <name>.casa;

    location / {
        proxy_pass http://127.0.0.1:<port>;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### Case B: Static Files on NAS

```nginx
server {
    listen 80;
    server_name <name>.casa;

    root /mnt/nas/webs/<name>.casa;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### Enable and Reload

```bash
# On maya:
sudo ln -sf /etc/nginx/sites-available/<name>.casa /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

# On pihole2 (via SSH):
ssh 192.168.1.54 "sudo ln -sf /etc/nginx/sites-available/<name>.casa /etc/nginx/sites-enabled/ && sudo nginx -t && sudo systemctl reload nginx"
```

---

## Step 3 — /etc/hosts on maya

```bash
echo "<target-IP> <name>.casa" | sudo tee -a /etc/hosts
```

> ⚠️ Required so that maya itself can resolve the domain (`mdns4_minimal` in nsswitch.conf blocks DNS for `.casa`).

---

## Step 4 — Card in portal.casa

Edit `/mnt/nas/webs/portal.casa/index.html`:

1. Increment the badge of the corresponding section (`Services`, `Infrastructure`, etc.)
2. Insert before the closing `</div></div>` of that section:

```html
<a class="card" href="http://<name>.casa" data-check="http://<name>.casa" data-name="<search keywords>">
  <div class="card-top"><div class="card-icon"><emoji></div><div class="dot checking"></div></div>
  <div class="card-name"><Visible Name></div>
  <div class="card-desc"><Brief description.></div>
  <div class="card-meta"><span class="card-url"><name>.casa</span><span class="badge"><host-machine></span></div>
</a>
```

**Available sections in portal.casa**: `Services`, `Infrastructure`, `Machines`

---

## Step 5 — Verification

```bash
# DNS resolves on Pi-holes
ssh 192.168.1.53 "nslookup <name>.casa 127.0.0.1 2>/dev/null | grep Address | tail -1"

# Nginx responds (bypassing maya DNS with Host header)
curl -s -H "Host: <name>.casa" http://<target-IP>/ | head -3

# Card in portal
curl -s http://portal.casa | grep -o '<name>.casa'
```

Security: expose the service only through nginx, open no extra ports, and apply least privilege to the configs you create. Security is a design input, not a later review step.

---

## Current .casa Domains (Reference)

| Domain | IP | Machine | Type |
|--------|----|---------|------|
| portal.casa | 192.168.1.54 | pihole2 | static NAS |
| ha.casa | 192.168.1.54 | pihole2 | reverse proxy |
| atlas.casa | 192.168.1.54 | pihole2 | reverse proxy |
| oficina.casa | 192.168.1.54 | pihole2 | static NAS |
| nas.casa | 192.168.1.54 | pihole2 | reverse proxy |
| trello.casa | 192.168.1.55 | maya | static NAS + API proxy |
| maya.casa | 192.168.1.55 | maya | static |
