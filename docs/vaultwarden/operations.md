# Operations

Day-to-day maintenance for Vaultwarden.

---

## Admin panel

The admin panel is at `https://vault.maktaba.home/admin`.

It is protected by `ADMIN_TOKEN` in `/srv/vaultwarden/.env` (not your Vaultwarden login password).

### Enable or rotate the token

```bash
openssl rand -base64 48
```

Set the output as `ADMIN_TOKEN` in `/srv/vaultwarden/.env`, then restart:

```bash
cd ~/git/projects/vaultwarden
docker compose restart vaultwarden
```

Leave `ADMIN_TOKEN` empty to disable the admin panel entirely.

---

## Health check

Run anytime:

```bash
~/git/projects/vaultwarden/healthcheck.sh
```

Checks: containers running, HTTPS alive endpoint, signups locked, latest backup exists.

---

## Backups

- **Schedule:** daily at 02:00 (backup sidecar cron)
- **Location:** `/srv/vaultwarden/backups/vw-backup-*.sqlite3`
- **Retention:** 14 most recent snapshots

### Manual backup

```bash
docker exec vaultwarden-backup /bin/sh /backup.sh
```

### Verify backup integrity

```bash
LATEST=$(ls -1t /srv/vaultwarden/backups/vw-backup-*.sqlite3 | head -n 1)
docker exec vaultwarden-backup sqlite3 "/backups/$(basename "$LATEST")" "PRAGMA integrity_check;"
```

Expected output: `ok`

### Restore from backup (test or disaster recovery)

**Stop Vaultwarden first** — never overwrite the live DB while it is running.

```bash
cd ~/git/projects/vaultwarden
docker compose stop vaultwarden

# Safety copy of current DB
cp /srv/vaultwarden/data/db.sqlite3 \
   /srv/vaultwarden/data/db.sqlite3.before-restore-$(date +%Y%m%d_%H%M%S)

# Restore (replace LATEST with your backup filename)
LATEST=$(ls -1t /srv/vaultwarden/backups/vw-backup-*.sqlite3 | head -n 1)
cp "$LATEST" /srv/vaultwarden/data/db.sqlite3

docker compose start vaultwarden
```

Then open `https://vault.maktaba.home` and confirm your vault loads.

To roll back, stop vaultwarden again and copy the `db.sqlite3.before-restore-*` file back.

---

## Trust Caddy local CA on other devices

Caddy uses an internal CA (`tls internal`) for `vault.maktaba.home` and `photos.maktaba.home` (same reverse proxy). Browsers show a warning; Immich and other apps usually **refuse** until the CA is trusted.

**Root certificate (in the Caddy data volume):**

```
/srv/vaultwarden/caddy/data/caddy/pki/authorities/local/root.crt
```

(Host path may be root-only; export with:  
`docker cp vaultwarden-caddy:/data/caddy/pki/authorities/local/root.crt ~/maktaba-home-ca.crt`)

Install that file as a **trusted CA** on each device. One install covers vault + photos.

### Linux (this VM)

```bash
docker cp vaultwarden-caddy:/data/caddy/pki/authorities/local/root.crt /tmp/maktaba-home-ca.crt
sudo cp /tmp/maktaba-home-ca.crt /usr/local/share/ca-certificates/maktaba-home-ca.crt
sudo update-ca-certificates
curl -fsS https://photos.maktaba.home/api/server/ping   # should work without -k
```

### macOS

1. Copy `maktaba-home-ca.crt` to the Mac (AirDrop, scp, etc.)
2. Double-click → Keychain Access → add to **System** keychain
3. Find the cert → Get Info → Trust → **Always Trust**

### Windows

1. Copy the `.crt` to the PC
2. `certmgr.msc` → Trusted Root Certification Authorities → Import

### Android (needed for Immich auto-backup)

1. Get `maktaba-home-ca.crt` onto the phone (USB, Drive, Messages, etc.)
2. Settings → Security → Encryption & credentials → **Install a certificate** → **CA certificate**
3. Select the file → confirm the warning (expected for a home CA)
4. Keep **Tailscale** connected
5. Open Immich → `https://photos.maktaba.home` → log in → enable backup  
   (Browser check: `https://vault.maktaba.home` should load without a warning)

### iOS

Settings → General → VPN & Device Management (or Profile Downloaded) after opening the `.crt` → Install → enable Full Trust for the root under Certificate Trust Settings.

After trusting the CA, HTTPS to vault and photos should work without warnings.

---

## Signups

New account registration is controlled by `SIGNUPS_ALLOWED` in `/srv/vaultwarden/.env`.

Keep it `false` after your initial account is created. To invite users later, temporarily set it to `true`, have them register, then set it back to `false` and restart vaultwarden.

---

## Optional later checklist

These are not required for normal use but worth doing when you have time:

- [ ] Trust Caddy local CA on every device that accesses Vaultwarden
- [ ] Run a backup restore test (see above) once to confirm backups are usable
- [ ] Set up remote access for devices without Tailscale — see [remote-access.md](remote-access.md)
- [ ] Configure SMTP in `/srv/vaultwarden/.env` if you want invitation emails
