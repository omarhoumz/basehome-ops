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
| [`docs/infrastructure/`](docs/infrastructure/) | Network topology, Proxmox, shared storage |

## Conventions

- **No secrets** in this repo — passwords, tokens, cookie files stay on-server only.
- When live docs change on a server, sync the matching file here the same day.
- Server-of-record for maktaba runtime docs: `/srv/maktaba/documentation/` (mirror of `docs/maktaba/`).
