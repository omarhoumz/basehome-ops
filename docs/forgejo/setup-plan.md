# Forgejo setup plan

| Field | Value |
| --- | --- |
| **Status** | `planned` |
| **Last updated** | 2026-09-07 |
| **Upstream** | [forgejo.org](https://forgejo.org/) · [docs](https://forgejo.org/docs/latest/) |
| **Topology** | [`docs/infrastructure/topology.md`](../infrastructure/topology.md) · [scan HTML](../infrastructure/topology.html) |

Self-hosted Git forge (community Gitea fork): repos, PRs, issues, packages. Private home alternative/complement to GitHub for `basehome-ops` and other homelab code. Matches **Vaultwarden / Immich** edge pattern (own `/srv` stack + Caddy `tls internal` + Tailscale-only).

---

## Decisions

| # | Decision | Why |
| --- | --- | --- |
| 1 | **Same Kubuntu VM** | Lightweight Go app; reuse Tailscale, dnsmasq, Docker, Caddy |
| 2 | **Own stack** under `/srv/forgejo` | Independent compose + DB lifecycle |
| 3 | **Project repo** `~/git/projects/forgejo` | Same layout as Immich / Homepage |
| 4 | **URL** `https://git.maktaba.home` | Familiar forge hostname on `*.maktaba.home` |
| 5 | **Postgres on local SSD** | Same invariant as Immich — never put DB on NFS |
| 6 | **HTTPS via vaultwarden-caddy `:443`** | Reuse local CA; validate vault `/alive` after reload |
| 7 | **Tailscale-only day one** | No ISP port-forward for HTTP or SSH |
| 8 | **Git over HTTPS first**; SSH later | Avoid host `:22` conflict; optional `:222` on Tailscale IP later |
| 9 | **No Actions runner day one** | Keep RAM/complexity low; add runner when CI is needed |
| 10 | **Disable open registration** | Single-admin / invite-only after first user |

---

## Resource budget

| Consumer | Cap | Notes |
| --- | --- | --- |
| `forgejo` | `1024M` | Web + git HTTP; usually well under this |
| `forgejo-db` (Postgres) | `512M` | Small personal forge |

**Sum of caps:** ~1.5 GiB. Stagger DB dumps away from Immich `02:30` / maktaba `02:00` — target **03:00**.

---

## Target shape

```
Kubuntu maktaba VM (192.168.11.141 / TS 100.68.38.53)
  ├─ /srv/maktaba/
  ├─ /srv/vaultwarden/         (Caddy :443)
  ├─ /srv/immich/
  ├─ /srv/homepage/            (planned)
  └─ /srv/forgejo/             ← NEW
       ├─ data/                (repos, config, attachments — LOCAL SSD)
       ├─ postgres/            (LOCAL SSD)
       └─ backups/             (pg dumps @ 03:00)

~/git/projects/forgejo/        ← NEW compose + healthcheck

Clients (Tailscale) → https://git.maktaba.home
                         → vaultwarden-caddy:443 → forgejo:3000
```

Repos stay on the VM root disk (71G). If git LFS / large binary repos grow, revisit a dedicated NFS path later — **not** for Postgres.

---

## Compose sketch (sanitized)

```yaml
services:
  forgejo:
    image: codeberg.org/forgejo/forgejo:11
    container_name: forgejo
    mem_limit: 1024M
    environment:
      USER_UID: 1000
      USER_GID: 1000
      FORGEJO__database__DB_TYPE: postgres
      FORGEJO__database__HOST: forgejo-db:5432
      FORGEJO__database__NAME: forgejo
      FORGEJO__database__USER: forgejo
      FORGEJO__database__PASSWD: ${FORGEJO_DB_PASSWORD}
      FORGEJO__server__DOMAIN: git.maktaba.home
      FORGEJO__server__ROOT_URL: https://git.maktaba.home/
      FORGEJO__server__HTTP_PORT: 3000
      FORGEJO__server__DISABLE_SSH: "true"   # day one; enable later with :222
      FORGEJO__service__DISABLE_REGISTRATION: "true"
    volumes:
      - /srv/forgejo/data:/data
    networks:
      - vaultwarden_default
      - forgejo_internal
    depends_on:
      - forgejo-db
    restart: unless-stopped

  forgejo-db:
    image: postgres:16-alpine
    container_name: forgejo-db
    mem_limit: 512M
    environment:
      POSTGRES_USER: forgejo
      POSTGRES_PASSWORD: ${FORGEJO_DB_PASSWORD}
      POSTGRES_DB: forgejo
    volumes:
      - /srv/forgejo/postgres:/var/lib/postgresql/data
    networks:
      - forgejo_internal
    restart: unless-stopped

networks:
  vaultwarden_default:
    external: true
  forgejo_internal:
    driver: bridge
```

Do **not** publish host `:3000` if proxied via Caddy. Keep `FORGEJO_DB_PASSWORD` in on-server `.env` (chmod 600) — never in basehome-ops git.

Caddy site (Vaultwarden edge):

```caddy
git.maktaba.home {
	tls internal
	reverse_proxy forgejo:3000
}
```

---

## Mitigations

### 1. Vaultwarden Caddy edge isolation
Same ritual as Immich/Homepage: `caddy validate` → reload → `curl -k -sf https://vault.maktaba.home/alive`.

### 2. Disk growth on VM SSD
Repos + LFS can fill the 71G root. Day one: personal/docs repos only; monitor with existing disk alerts in maktaba `healthcheck.sh`. Move bulk to NFS only if needed (data volume — still keep Postgres local).

### 3. Backup / restore
Nightly `pg_dump` of `forgejo` DB + rsync or tar of `/srv/forgejo/data` (repos are not only in Postgres). Retention: keep last 7. Restore drill before calling the service critical.

### 4. Registration lockdown
Create the admin account on first visit, then keep `DISABLE_REGISTRATION=true` (or invite-only).

### 5. Relationship to GitHub
Public mirrors (e.g. `basehome-ops` on GitHub) can stay. Forgejo is for private/home-only repos and a local clone source over Tailscale — not a forced migration.

---

## Phased rollout

### Phase 1 — Project + runtime dirs
1. Create `~/git/projects/forgejo` with `docker-compose.yml`, `.env.example`, `healthcheck.sh`, `backup.sh`, `.gitignore`.
2. `mkdir -p /srv/forgejo/{data,postgres,backups}` with UID/GID 1000 ownership.

### Phase 2 — Stack up
1. Join `vaultwarden_default` + internal DB network; no host port bind for HTTP.
2. `docker compose up -d`; complete web install / admin user; confirm registration disabled.
3. Verify `mem_limit` via `docker stats`.

### Phase 3 — Edge & DNS
1. Add `git.maktaba.home` to Vaultwarden `Caddyfile`; validate + reload; check vault `/alive`.
2. Confirm HTTPS over Tailscale; push a test repo over HTTPS.
3. Update REGISTRY `planned` → `live` the same day.

### Phase 4 — Ops hygiene
1. Schedule Postgres (+ data) backup at **03:00**.
2. Document ops in `docs/forgejo/operations.md`.
3. Optional: add link on Homepage; enable SSH on Tailscale `:222`; add Actions runner on a separate compose when CI is needed.
