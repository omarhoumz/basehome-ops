# Immich — operations

Day-to-day operations and maintenance for Immich on the Kubuntu VM.

Friend allowlist, phone Access headers, and service-token rotation: [public-access-runbook.md](./public-access-runbook.md).  
Design: [2026-09-08-public-access-design.md](./2026-09-08-public-access-design.md).

---

## Service URLs

| URL | Component | Notes |
| --- | --- | --- |
| `https://photos.homz.fyi` | Immich Web / API (public) | Cloudflare Tunnel `maktaba-immich` + Access (OTP / service token). Household phones and friends. |
| `https://photos.maktaba.home` | Immich Web / API (private) | Proxied via `vaultwarden-caddy` with `tls internal`. Prefer for **admin**. Tailscale. |
| `http://127.0.0.1:2283` | Direct local port | Internal host-only binding |

**Vaultwarden** (`https://vault.maktaba.home`) stays Tailscale-only and is **not** on the Immich tunnel.

---

## Start / stop

```bash
cd ~/git/projects/immich
docker compose ps
docker compose up -d
docker compose stop
docker compose down    # container state destroyed; persistent data in /srv/immich and NFS
```

Public edge connector: `immich_cloudflared` (same compose project). Tunnel token: `/srv/immich/secrets/tunnel_token.txt` (not in git).

---

## Health check

Run anytime:

```bash
~/git/projects/immich/healthcheck.sh
```

Checks:
1. Core containers running (`immich_server`, `immich_machine_learning`, `immich_postgres`, `immich_redis`).
2. `immich_cloudflared` running (public tunnel connector).
3. Live memory usage vs enforced caps.
4. Storage write access (`UPLOAD_LOCATION` on WD Red, `DB_DATA_LOCATION` on local SSD).
5. Local API ping (`http://127.0.0.1:2283/api/server/ping`).
6. Neighbor safety (`https://vault.maktaba.home/alive`).

After any Caddy change on the Vaultwarden edge, re-check vault `/alive`.

---

## Backups

Immich data is divided into two separate storage tiers:
1. **Database & metadata (local SSD):** Stored in `/srv/immich/postgres`. Contains album definitions, users, tags, face embeddings, and asset pointers.
2. **Bulk photos & videos (WD Red):** Stored in `/srv/immich/library` (NFS to USB WD Red).

### Database backup

Run manual backup:

```bash
~/git/projects/immich/backup.sh
```

- **Output:** `/srv/immich/backups/immich-db-<timestamp>.sql.gz`
- **Retention:** Automatically keeps the last 7 days of snapshots.
- **Scheduled cron:** Recommended at `02:30` (staggered vs Vaultwarden/Maktaba at `02:00`):
  ```cron
  30 2 * * * /home/basehome/git/projects/immich/backup.sh >> /var/log/immich-backup.log 2>&1
  ```

### Database restore

1. Stop Immich services (except database):
   ```bash
   cd ~/git/projects/immich
   docker compose stop immich-server immich-machine-learning
   ```

2. Restore database from backup:
   ```bash
   gunzip -c /srv/immich/backups/immich-db-YYYYMMDD_HHMMSS.sql.gz | \
     docker exec -i immich_postgres psql -U postgres -d immich
   ```

3. Restart stack:
   ```bash
   docker compose up -d
   ./healthcheck.sh
   ```

---

## Mobile client setup

### Public URL (household default)

See [public-access-runbook.md](./public-access-runbook.md) — Server URL `https://photos.homz.fyi` plus `CF-Access-Client-Id` / `CF-Access-Client-Secret` headers. No Tailscale or Caddy CA required.

### Private URL (admin / Tailscale)

1. Install Immich app on iOS or Android.
2. Ensure device is connected to **Tailscale**.
3. Set Server URL: `https://photos.maktaba.home`.
4. If using self-signed internal CA, install the Caddy root as a trusted CA on the device  
   (`docker cp vaultwarden-caddy:/data/caddy/pki/authorities/local/root.crt ~/maktaba-home-ca.crt` — see [vaultwarden operations](../vaultwarden/operations.md)).
5. Log in with user credentials created during initial admin setup.

---

## Cloudflare Access & rate limits

Managed in Cloudflare Zero Trust / zone `homz.fyi` (no resource IDs in git):

| Piece | Name / scope | Role |
| --- | --- | --- |
| Tunnel | `maktaba-immich` | Public hostname `photos.homz.fyi` → `http://immich_server:2283` only |
| Access app | `photos.homz.fyi` (all paths) | One-time PIN IdP |
| Policy | `immich-household` | Permanent household emails |
| Policy | `immich-guests` | Temporary friend emails (add/remove) |
| Service token | `immich-mobile` | Phone headers; rotate per runbook |
| Rate limit | Host `photos.homz.fyi` | Zone WAF rate-limiting rule (Free often allows 1 rule — keep it Immich-scoped). Suggested start: ~100–300 req/min/IP, Block. Record the live expression/threshold in this table when set. |

Large uploads via the public URL may fail (~100MB Cloudflare body limit) — see runbook.

---

## Upgrades

1. Check Immich release notes on GitHub for any breaking changes:
   https://github.com/immich-app/immich/releases
2. Update stack:
   ```bash
   cd ~/git/projects/immich
   docker compose pull
   docker compose up -d
   ./healthcheck.sh
   ```
