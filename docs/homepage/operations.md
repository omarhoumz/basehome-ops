# Homepage operations

Start-page dashboard at `https://home.maktaba.home` (Tailscale + Caddy `tls internal`).

| Field | Value |
| --- | --- |
| **URL** | `https://home.maktaba.home` |
| **Compose** | `~/git/projects/homepage/` |
| **Config** | `/srv/homepage/config/` (YAML) |
| **Edge** | `vaultwarden-caddy` → `homepage:3000` on `vaultwarden_default` |
| **Access** | Tailscale-only (no public tunnel) |

---

## Health check

```bash
~/git/projects/homepage/healthcheck.sh
```

Checks: container running, HTTPS endpoint, vault neighbor `/alive`.

---

## Start / stop

```bash
cd ~/git/projects/homepage
docker compose up -d
docker compose down
```

---

## Edit links / widgets

1. Edit YAML under `/srv/homepage/config/` (or sync from `~/git/projects/homepage/config/`).
2. Homepage reloads most config automatically; recreate the container after compose changes (e.g. Docker socket).

| File | Purpose |
| --- | --- |
| `settings.yaml` | Title, theme, layout |
| `services.yaml` | App tiles + optional `server`/`container` for Docker status |
| `docker.yaml` | Docker socket target (`local`) |
| `bookmarks.yaml` | Extra bookmarks |
| `widgets.yaml` | Search, datetime, system resources, **wdred** disk |

---

## Caddy changes

After editing `~/git/projects/vaultwarden/Caddyfile`:

```bash
cd ~/git/projects/vaultwarden
docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
curl -k -sf https://vault.maktaba.home/alive
```

---

## Scope

- Docker socket mounted **read-only** for container status on tiles (Jellyfin, TA, Immich, Vaultwarden).
- Widgets: DuckDuckGo search, datetime, container resources, **wdred** disk (`/mnt/wdred` ← NFS `shared`, ~3.7T).
- Service API widgets (Immich/Jellyfin stats) later via `HOMEPAGE_VAR_*` in `.env` (chmod 600).
- Harden later with [docker-socket-proxy](https://github.com/Tecnativa/docker-socket-proxy) if desired.
- Forgejo tile points at planned `git.maktaba.home` until Forgejo is live.

### wdred disk widget

Compose mounts host `/mnt/wdred/shared` → container `/mnt/wdred:ro` (same 3.7T USB as media + Immich library). Usage shown is the whole NFS export, not a separate partition.
