# basehome-ops

**Docs-only** repo for home infrastructure — same idea as [tibr-ops](https://github.com/omarhoumz/tibr-ops), but for the homelab.

Live configs, compose files, and secrets stay **on the servers** (`/srv/maktaba`, `/srv/vaultwarden`, Proxmox host). This repo is the sanitized, versioned copy for humans and agents.

## Start here

| Doc | When |
| --- | --- |
| [`docs/TASKS.md`](docs/TASKS.md) | Open work across home infra |
| [`projects/REGISTRY.md`](projects/REGISTRY.md) | What runs where (IPs, URLs, status) |
| [`docs/maktaba/handoff.md`](docs/maktaba/handoff.md) | Pick up a maktaba session |
| [`AGENTS.md`](AGENTS.md) | Rules for agents working on home ops |

## Docs layout

| Path | What |
| --- | --- |
| [`docs/maktaba/`](docs/maktaba/) | Media stack (Jellyfin, TubeArchivist, USB/NFS, DNS) |
| [`docs/vaultwarden/`](docs/vaultwarden/) | Password manager ops + remote access |
| [`docs/immich/`](docs/immich/) | Photo library — [operations](docs/immich/operations.md) |
| [`docs/homepage/`](docs/homepage/) | Start-page dashboard — [operations](docs/homepage/operations.md) |
| [`docs/forgejo/`](docs/forgejo/) | Self-hosted Git forge (planned) — [setup-plan](docs/forgejo/setup-plan.md) |
| [`docs/infrastructure/`](docs/infrastructure/) | Topology SoT + network pointers |
| [`docs/popular-homelab-projects.md`](docs/popular-homelab-projects.md) | Community top-10 + mentions (candidates, not deploy plan) |

### Topology (source of truth)

Edit [`docs/infrastructure/topology.yaml`](docs/infrastructure/topology.yaml), then:

```bash
python3 scripts/render-topology.py
```

| Output | For |
| --- | --- |
| [`topology.md`](docs/infrastructure/topology.md) | Humans + agents |
| [`topology.html`](docs/infrastructure/topology.html) | Scan page |

## Conventions

- **No secrets** in this repo — passwords, tokens, cookie files stay on-server only.
- When live docs change on a server, sync the matching file here the same day.
- Server-of-record for maktaba runtime docs: `/srv/maktaba/documentation/` (mirror of `docs/maktaba/`).
