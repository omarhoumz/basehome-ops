# Immich — public access runbook

Procedures for `https://photos.homz.fyi` (Cloudflare Tunnel + Access).  
Private admin URL stays `https://photos.maktaba.home` (Tailscale).  
**Vaultwarden is Tailscale-only** and must never be added to this tunnel.

Related: [operations.md](./operations.md) · [design](./2026-09-08-public-access-design.md)

---

## Share with a friend

Temporary guest access is email allowlist only. Immich registration stays off.

1. Cloudflare Zero Trust → Access → Applications → Immich app (`photos.homz.fyi`).
2. Open the **immich-guests** Allow policy.
3. Add the friend’s email to the include list; save.
4. Create or copy an Immich share / album link that uses `https://photos.homz.fyi` (not the Tailscale host).
5. Send the link. Friend opens it → Cloudflare Access email → One-time PIN → Immich.
6. When sharing is done: remove their email from **immich-guests** and save. Confirm they are blocked (incognito).

Do not put guest emails on the permanent **immich-household** policy.

---

## Phone setup (household)

No Tailscale or local CA required for the public URL.

1. Install the Immich mobile app.
2. Server URL: `https://photos.homz.fyi`
3. Advanced → custom headers (from Vaultwarden secure note for the Immich mobile service token):
   - `CF-Access-Client-Id` → Client ID
   - `CF-Access-Client-Secret` → Client Secret
4. Log in with the household Immich user (created by admin; registration is off).

Admin / rescue path without public Access: Tailscale + `https://photos.maktaba.home` (see [operations.md](./operations.md)).

---

## Rotate the Access service token

Use when a phone is lost, a secret may have leaked, or on a regular rotation cadence.

1. Zero Trust → Access → Service auth → Create a new service token (e.g. `immich-mobile` successor).
2. Store the new Client ID + Client Secret in Vaultwarden; keep the old note until phones are updated.
3. Attach the new token to the Immich Access application (Service Auth / Bypass policy), same as the existing mobile token.
4. Update every household phone: replace `CF-Access-Client-Id` / `CF-Access-Client-Secret`.
5. Confirm backup/sync works on each device.
6. Revoke / delete the old service token in Cloudflare.
7. Remove the old secret from Vaultwarden (or mark revoked).

Never commit Client ID/Secret to git. Tunnel token stays under `/srv/immich/secrets/` + Vaultwarden only.

---

## Large videos (~100MB Cloudflare limit)

Cloudflare’s HTTP proxy (including Tunnel) caps request bodies around **~100MB**. Uploading a large video via `https://photos.homz.fyi` may fail.

- **Household default:** keep using the public URL; accept the limit.
- **Optional rescue (ops / admin):** on Tailscale, upload via `https://photos.maktaba.home` instead. Not required on phones for normal day-to-day use.

---

## Admin habit

Prefer **`https://photos.maktaba.home`** for Immich admin UI and heavy maintenance.

Use the public URL when testing Access, guest flows, or household phone behaviour. Keep a single Immich admin user; do not open registration.
