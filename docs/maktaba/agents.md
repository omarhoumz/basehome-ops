# Agent notes — maktaba

Live stack: `/srv/maktaba`. Docs mirror: `basehome-ops/docs/maktaba/`.

## Invariants

- Jellyfin mounts `data/media` **read-only** (NFS-backed)
- `TA_HOST=http://yt.maktaba.home` (no `:8000`)
- Plugin → `http://tubearchivist:8000` (Docker DNS)
- ES password secret shared between `tubearchivist` and `archivist-es`
- Never put `.bak` files in `/etc/dnsmasq.d/`
- Tailscale IP in dnsmasq: `100.68.38.53`
- NFS requires Proxmox USB mounted + nfs-server active
- Before USB unplug: stop TA + Jellyfin, umount NFS (see usb-wdred-runbook)

## Safe commands

```bash
cd /srv/maktaba
docker compose ps
./scripts/healthcheck.sh
./scripts/update-safe.sh ta|jellyfin|all
dig +short @100.68.38.53 yt.maktaba.home
```

## YouTube cookies

Stored in Redis. Re-import when 403 returns — see [operations.md](./operations.md).

## ES index failures

Root disk ≥95% → ES read-only. Free space, clear block, retry download. See server `/srv/maktaba/README.md`.
