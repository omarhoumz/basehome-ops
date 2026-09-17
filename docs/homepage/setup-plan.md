# Homepage setup plan

| Field | Value |
| --- | --- |
| **Status** | `live` |
| **Last updated** | 2026-09-17 |
| **Upstream** | [gethomepage/homepage](https://github.com/gethomepage/homepage) · [docs](https://gethomepage.dev) |
| **Topology** | [`docs/infrastructure/topology.md`](../infrastructure/topology.md) · [scan HTML](../infrastructure/topology.html) |

YAML-configured application dashboard (start page) for the homelab: bookmarks, Docker discovery, and widgets for Jellyfin, Immich, Vaultwarden, TubeArchivist, and host stats. Matches **Vaultwarden / Immich** edge pattern (own `/srv` stack + Caddy `tls internal` + Tailscale-only).

---

## Decisions

| # | Decision | Why |
| --- | --- | --- |
| 1 | **Same Kubuntu VM** | Homepage is lightweight; reuse Tailscale, dnsmasq, Docker, Caddy |
| 2 | **Own stack** under `/srv/homepage` | Independent compose lifecycle; config survives rebuilds |
| 3 | **Project repo** `~/git/projects/homepage` | Same layout as Immich / Vaultwarden (compose + healthcheck in git) |
| 4 | **URL** `https://home.maktaba.home` | Clear start-page hostname on existing `*.maktaba.home` |
| 5 | **HTTPS via vaultwarden-caddy `:443`** | Reuse local CA; validate with vault `/alive` after reload |
| 6 | **Tailscale-only day one** | No ISP port-forward; private dashboard |
| 7 | **Docker socket read-only for tile status** | Day-one static links; Phase 4 mounts socket `:ro` for container status. Prefer socket proxy later if hardening |
| 8 | **No NFS needed** | Config + cache are tiny; stay on VM SSD |

---

## Resource budget

Homepage is a single Node container — typically **&lt;256 MiB** RAM idle.

| Consumer | Cap | Notes |
| --- | --- | --- |
| `homepage` | `512M` | Hard cgroup ceiling; still tiny vs Immich/ES |

No Postgres/Redis. Neighbor risk is mainly a bad Caddy reload (same as Immich) — always hit `https://vault.maktaba.home/alive` after changes.

---

## Target shape

```
Kubuntu maktaba VM (192.168.11.141 / TS 100.68.38.53)
  ├─ /srv/maktaba/             (unchanged)
  ├─ /srv/vaultwarden/         (unchanged: Caddy :443)
  ├─ /srv/immich/              (unchanged)
  └─ /srv/homepage/            ← NEW
       └─ config/              (YAML: services, widgets, settings, bookmarks)

~/git/projects/homepage/       ← NEW compose + healthcheck

Clients (Tailscale) → https://home.maktaba.home
                         → vaultwarden-caddy:443 → homepage:3000
```

---

## Config sketch (sanitized)

`/srv/homepage/config/settings.yaml` — set `title`, layout, theme as preferred.

`/srv/homepage/config/services.yaml` — day-one links (widgets optional later):

```yaml
- Media:
    - Jellyfin:
        href: http://player.maktaba.home
        description: Movies / TV
        icon: jellyfin.png
    - TubeArchivist:
        href: http://yt.maktaba.home
        description: YouTube archive
        icon: tubearchivist.png

- Photos & secrets:
    - Immich:
        href: https://photos.maktaba.home
        description: Photo library
        icon: immich.png
    - Vaultwarden:
        href: https://vault.maktaba.home
        description: Passwords
        icon: vaultwarden.png

- Dev:
    - Forgejo:
        href: https://git.maktaba.home
        description: Git forge
        icon: forgejo.png
```

Compose essentials:

```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    mem_limit: 512M
    environment:
      HOMEPAGE_ALLOWED_HOSTS: home.maktaba.home
      PUID: 1000
      PGID: 1000
    volumes:
      - /srv/homepage/config:/app/config
    networks:
      - vaultwarden_default
    restart: unless-stopped

networks:
  vaultwarden_default:
    external: true
```

Do **not** publish `:3000` on the host LAN if proxied via Caddy on the shared Docker network.

Caddy site (Vaultwarden edge):

```caddy
home.maktaba.home {
	tls internal
	reverse_proxy homepage:3000
}
```

---

## Mitigations

### 1. Vaultwarden Caddy edge isolation
Same ritual as Immich: `caddy validate` → reload → `curl -k -sf https://vault.maktaba.home/alive`.

### 2. Docker socket exposure
Mounting the socket (even `:ro`) grants broad Docker API access inside the container. Day one: accept for discovery convenience, or skip the mount and use static links only. Hardening later: [docker-socket-proxy](https://github.com/Tecnativa/docker-socket-proxy) with a tight allowlist.

### 3. Secrets in widgets
API keys for Immich / Vaultwarden widgets use `HOMEPAGE_VAR_*` / `HOMEPAGE_FILE_*` env — never commit keys into YAML in git. Keep secrets on-server under `/srv/homepage` or Compose env files (chmod 600).

---

## Phased rollout

### Phase 1 — Project + runtime dirs
1. Create `~/git/projects/homepage` with `docker-compose.yml`, `.env.example`, `healthcheck.sh`, `.gitignore`.
2. `mkdir -p /srv/homepage/config` and seed minimal YAML (`settings.yaml`, `services.yaml`, `widgets.yaml`, `bookmarks.yaml`, `docker.yaml` if using socket).

### Phase 2 — Stack up
1. Join `vaultwarden_default` network; do not bind host port 3000.
2. `docker compose up -d`; confirm container healthy and `mem_limit` visible in `docker stats`.

### Phase 3 — Edge & DNS
1. Add `home.maktaba.home` to Vaultwarden `Caddyfile`; validate + reload; check vault `/alive`.
2. Confirm HTTPS over Tailscale.
3. Update REGISTRY `planned` → `live` the same day.

### Phase 4 — Widgets + Docker discovery
1. Mount Docker socket `:ro`; add `docker.yaml` + `server`/`container` on live tiles.
2. Search / datetime / resources widgets (no API keys).
3. Document ops in `docs/homepage/operations.md`.
4. Later: service API widgets via `HOMEPAGE_VAR_*`; prefer docker-socket-proxy for hardening.
