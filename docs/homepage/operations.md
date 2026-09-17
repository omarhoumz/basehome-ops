# Homepage operations

Start-page dashboard at `https://home.maktaba.home` (Tailscale + Caddy `tls internal`).

| Field | Value |
| --- | --- |
| **URL** | `https://home.maktaba.home` |
| **Compose** | `~/git/projects/homepage/` |
| **Config** | `/srv/homepage/config/` (YAML) |
| **Edge** | `vaultwarden-caddy` → `homepage:3000` on `vaultwarden_default` |
| **Access** | Tailscale-only (no public tunnel) |
| **Docker API** | `homepage_dockerproxy` (Tecnativa) → Homepage; **no** raw socket in Homepage |

---

## Health check

```bash
~/git/projects/homepage/healthcheck.sh
```

Checks: homepage + dockerproxy running, HTTPS endpoint, vault neighbor `/alive`.

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
2. Homepage reloads most config automatically; recreate after compose changes.

| File | Purpose |
| --- | --- |
| `settings.yaml` | Title, theme, layout |
| `services.yaml` | App tiles + `server`/`container` for Docker status |
| `docker.yaml` | Points at `dockerproxy:2375` (not host socket) |
| `bookmarks.yaml` | Extra bookmarks |
| `widgets.yaml` | Search, datetime, system resources, **wdred** disk |

---

## Docker socket proxy

Only `homepage_dockerproxy` mounts `/var/run/docker.sock:ro`. Homepage talks to it over the Docker network (`CONTAINERS=1`, `POST=0`). Port **2375 is not published** on the host.

```yaml
# docker.yaml
local:
  host: dockerproxy
  port: 2375
```

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

- Container status via **docker-socket-proxy** (not a direct socket mount on Homepage).
- Widgets: DuckDuckGo search, datetime, resources, **wdred** disk (`/mnt/wdred` ← NFS `shared`, ~3.7T).
- Service API widgets: see [2026-09-17-service-api-widgets-plan.md](./2026-09-17-service-api-widgets-plan.md).
- Forgejo tile points at planned `git.maktaba.home` until Forgejo is live.

### wdred disk widget

Compose mounts host `/mnt/wdred/shared` → container `/mnt/wdred:ro` (same 3.7T USB as media + Immich library). Usage shown is the whole NFS export, not a separate partition.
