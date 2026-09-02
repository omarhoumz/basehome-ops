# Maktaba — operations

Home media stack on Kubuntu VM `maktaba`. Live files: `/srv/maktaba`.

| URL | Service |
| --- | --- |
| http://player.maktaba.home | Jellyfin |
| http://yt.maktaba.home | TubeArchivist |
| http://192.168.11.141 | Jellyfin (IP alias) |

## Start / stop

```bash
cd /srv/maktaba
docker compose ps
docker compose up -d
docker compose down    # data persists in ./data
```

## Health check

```bash
cd /srv/maktaba
./scripts/healthcheck.sh
```

Disk section warns at 85% root usage, fails at 95%. Checks NFS media mount.

## Updates

**TubeArchivist + Elasticsearch** (read release notes first):

```bash
cd /srv/maktaba
./scripts/update-safe.sh ta
```

**Jellyfin:**

```bash
./scripts/update-safe.sh jellyfin
```

**Caddy** after editing `Caddyfile`:

```bash
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

## YouTube cookies (403 errors)

1. Export `www.youtube.com_cookies.txt` from browser (Get cookies.txt LOCALLY).
2. Import:

```bash
docker cp ~/Downloads/www.youtube.com_cookies.txt tubearchivist:/tmp/yt_cookies.txt
docker exec tubearchivist python manage.py shell -c "
from pathlib import Path
from appsettings.src.config import AppConfig
from download.src.yt_dlp_base import CookieHandler
CookieHandler(AppConfig().config).set_cookie(Path('/tmp/yt_cookies.txt').read_text())
"
```

Exit node is **not** used for downloads.

## Backups

```bash
cd /srv/maktaba
./scripts/backup.sh
```

Backs up config/secrets/state — not full media bytes (media is on USB NFS).

## Troubleshooting

**`failed to add item to index`** — VM disk full; ES locked read-only. Free space, then clear ES block (see live `/srv/maktaba/README.md` on server).

**DNS broken after reboot** — [dnsmasq-tailscale-boot.md](./dnsmasq-tailscale-boot.md)

**NFS media missing** — on Proxmox: `mount /mnt/pve/wdred`; on maktaba: `mount /srv/maktaba/data/media`

## More docs

- [handoff.md](./handoff.md) — session pickup
- [usb-wdred-runbook.md](./usb-wdred-runbook.md) — USB storage
- [agents.md](./agents.md) — agent invariants
