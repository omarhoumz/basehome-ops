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

Caddy uses an internal CA (`tls internal`) for `vault.maktaba.home`. Browsers on devices that have not trusted this CA will show a certificate warning.

The root certificate lives on the server at:

```
/srv/vaultwarden/caddy/data/pki/authorities/local/root.crt
```

Copy it to each client device and install it as a trusted root CA.

### Linux (this VM — already trusted by Caddy on first start)

```bash
sudo cp /srv/vaultwarden/caddy/data/pki/authorities/local/root.crt \
        /usr/local/share/ca-certificates/vaultwarden-local.crt
sudo update-ca-certificates
```

### macOS

1. Copy `root.crt` to the Mac (AirDrop, scp, etc.)
2. Double-click → Keychain Access → add to System keychain
3. Find the cert → Get Info → Trust → Always Trust

### Windows

1. Copy `root.crt` to the PC
2. Run `certmgr.msc` → Trusted Root Certification Authorities → Import

### Android / iOS

Install the cert via Settings → Security → Install certificate (exact path varies by OS version). You may also need to add `vault.maktaba.home` to DNS or `/etc/hosts` pointing at the server IP.

After trusting the CA, `https://vault.maktaba.home` should load without warnings.

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
