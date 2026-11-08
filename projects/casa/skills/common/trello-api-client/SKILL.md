---
name: trello-api-client
description: >
  CLI to manage the .CASA intranet Trello clone (boards, lists, cards, custom fields)
  and its integration with SDD, Atlas, and Portal.
  Trigger: "trello board", "trello card", "trello list", "create trello board",
  "move card", "add card", "trello new", "move card".
license: Apache-2.0
metadata:
  author: Felipe Pérez
  version: "1.1"
---

## When to Use

Use when managing the intranet `.CASA` Trello clone: create/move boards, lists or cards,
define custom fields, or sync the SDD cycle with a board.

## Scope Guard

Skill **environment-bound**: assumes specific local infrastructure and is NOT portable.

- Native backend (systemd `poc-trello`) on **maya (192.168.1.55)**, port **3002**.
- Requires the `trello` binary authenticated (`trello login`) with token in `~/.trello/`.
- Depends on native PostgreSQL (`poc_trello`) and nginx (`trello.casa`).

Outside that environment, this skill does not apply.

## Service

```bash
sudo systemctl status poc-trello       # status (🟢 LISTEN on :3002, native, NOT Docker)
sudo journalctl -u poc-trello -f       # real-time logs
```

## Quick Start

```bash
trello login                                              # 1. authenticate
trello board create "My Project" --desc "Development"     # 2. create board
trello list create <board-id> "In Progress"              # 3. create column
trello card create <list-id> "Implement something"       # 4. create card
trello card move-to <card-id> --column "In Progress"     # 5. move card
```

## References

| File | Content |
|------|---------|
| [`references/commands.md`](references/commands.md) | All CLI commands (auth, boards, lists, cards, fields, integrations) + config |
| [`references/api-endpoints.md`](references/api-endpoints.md) | REST endpoints, bodies and error codes |
| [`references/models.md`](references/models.md) | Data models (Board, Card, CustomField…) |
| [`references/architecture.md`](references/architecture.md) | systemd service, nginx, PostgreSQL, env, diagnostics |
| [`references/sdd-integration.md`](references/sdd-integration.md) | SDD phases ↔ columns mapping and sync rules |
| [`references/examples.md`](references/examples.md) | Complete flows (SDD and manual) |

## Quick Troubleshooting

| Problem | Command |
|---------|---------|
| Service down | `sudo systemctl restart poc-trello` |
| View logs | `sudo journalctl -u poc-trello -n 50` |
| Port occupied | `sudo ss -tlnp \| grep 3002` |
| API 404 | `curl http://localhost:3002/health` |
| Nginx failed | `sudo nginx -t && sudo systemctl reload nginx` |

Extended detail in [`references/architecture.md`](references/architecture.md#troubleshooting).

## System References

- **Swagger UI:** http://localhost:3002/api-docs
- **OpenAPI:** `/mnt/nas/sources/pocs/poc-trello/backend/src/openapi/openapi.yaml`
- **Code:** `/mnt/nas/sources/pocs/poc-trello/`
- **Service:** `/etc/systemd/system/poc-trello.service`

## Model routing hints

- preferred agent: devops
- preferred model: ollama/qwen3.6:27b
- routing intent: hint only; the skill must not switch models directly
