# Immich — Cloudflare Access setup checklist

**Who:** human in Cloudflare Zero Trust dashboard  
**App:** `photos.homz.fyi` (all paths)  
**Already done:** tunnel `maktaba-immich` healthy; `https://photos.homz.fyi/api/server/ping` → 200 `pong`; `cloudflared` in Immich compose.

Phone headers / service-token usage: [public-access-runbook.md](./public-access-runbook.md).  
Do **not** put Client ID/Secret in this repo — Vaultwarden only. Rate limiting comes later.

---

## Checklist (Zero Trust only)

1. **One-time PIN IdP** (UI moved)  
   Open [Cloudflare One](https://one.dash.cloudflare.com/) → **Integrations** → **Identity providers**.  
   Under **Your identity providers** → **Add new identity provider** → **One-time PIN**.  
   (Old path `Settings → Authentication → Login methods` is gone / renamed.)

   Direct docs: https://developers.cloudflare.com/cloudflare-one/integrations/identity-providers/one-time-pin/

2. **Self-hosted Access application** (UI updated ~2026)  
   Access controls → Applications → Create new application → **Self-hosted and private**.  
   When asked how to connect / destination type, choose **Public DNS** (not Private destinations, Workers, or Service auth).  
   Add public hostname: subdomain `photos`, domain `homz.fyi` (path empty = all paths).  
   Session duration: e.g. 24h.  
   Identity providers: One-time PIN.

3. **Allow policy — `immich-household`**  
   On the app, add an Allow policy named `immich-household`.  
   Include rule: emails in selector → **fill your household emails here** (permanent allowlist).  
   Action: Allow. Order: above guests.

4. **Allow policy — `immich-guests`**  
   Add Allow policy named `immich-guests`.  
   Include: emails in selector — **may start empty**; add/remove friend emails when sharing.  
   Action: Allow.

5. **Service token — `immich-mobile`**  
   Access → Service auth → Create service token named `immich-mobile`.  
   Copy Client ID + Client Secret into a **Vaultwarden** secure note (never git).  
   On the Immich Access app, add a **Service Auth** / Bypass policy that accepts this token (so mobile can skip OTP).  
   Policy order: service-token policy must apply for clients sending the headers.

6. **Verify**  
   - Browser with an **unlisted** email → Access blocks (no Immich UI).  
   - Allowlisted email → OTP email → PIN works → Immich loads on `https://photos.homz.fyi`.

7. **Done signal**  
   When steps 1–6 are complete, reply: **access ready**

---

## Out of scope here

- Rate limits (later)  
- Immich user creation / registration (ops elsewhere)  
- Tunnel or compose changes (already done)
