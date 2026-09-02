# Agents — home infrastructure

**Checklist and registry live in basehome-ops — not scattered across product folders.**

## Start here

1. Read [`docs/TASKS.md`](docs/TASKS.md) for open items.
2. Confirm the service row in [`projects/REGISTRY.md`](projects/REGISTRY.md).
3. For maktaba sessions: read [`docs/maktaba/handoff.md`](docs/maktaba/handoff.md) first.
4. Execute on the **live server** — this repo has docs only, no compose or secrets.

## Who owns what

| Owner | Responsibility |
| --- | --- |
| **basehome-ops** | Registry, runbooks, handoff, architecture — no app code |
| **`/srv/maktaba`** | Maktaba docker stack, scripts, secrets, live documentation mirror |
| **`/srv/vaultwarden`** | Vaultwarden compose + data (git repo may live under `~/git/projects/vaultwarden`) |
| **Proxmox (`pve`)** | USB disk mount, NFS exports, VM backups on `wdred` storage |

## Don’t

- Commit secrets, `.env`, cookie files, or backup dumps.
- Edit compose on-server without updating docs here when behavior changes.
- Remount Jellyfin `data/media` read-write.
- Put `.bak` files in `/etc/dnsmasq.d/`.

## Maktaba quick invariants

- Live stack: `/srv/maktaba` (not `~/git/projects/maktaba`)
- `TA_HOST=http://yt.maktaba.home` (no `:8000`)
- Plugin → `http://tubearchivist:8000` (Docker DNS)
- Media: NFS from Proxmox `192.168.11.123:/mnt/pve/wdred/maktaba/media`
- Exit node: **off** on maktaba; YouTube uses browser cookies in Redis
- Tailscale split DNS for `maktaba.home` → nameserver `100.68.38.53`
