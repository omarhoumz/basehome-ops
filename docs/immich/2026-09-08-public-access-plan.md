# Immich public access — implementation plan

> **For agentic workers:** Implement task-by-task. Steps use checkbox (`- [ ]`) syntax. Do not publish Vaultwarden. Do not open Immich registration.

**Goal:** Expose Immich at `https://photos.homz.fyi` via Cloudflare Tunnel + Access (OTP + service tokens) + rate limits, keep `photos.maktaba.home` for admin, document runbooks.

**Architecture:** `cloudflared` joins the Immich Docker network and forwards only to `immich_server:2283`. Cloudflare Access gates the public hostname. Private Caddy path unchanged.

**Tech stack:** Docker Compose, Cloudflare Zero Trust (Tunnel + Access OTP + service tokens), existing Immich + vaultwarden-caddy on maktaba.

**Spec:** [2026-09-08-public-access-design.md](./2026-09-08-public-access-design.md)

## Global constraints

- Zone: `homz.fyi` (already on Cloudflare)
- Public hostname: `photos.homz.fyi` only on this tunnel
- Private URL: `https://photos.maktaba.home` remains for admin
- Immich registration stays **off**
- Tunnel must never route Vaultwarden or other services
- Secrets (`tunnel token`, service token) never committed; store in Vaultwarden + files under `/srv/immich/secrets/`
- Large uploads via public URL may fail (~100MB CF limit) — accepted; document only
- After any Caddy change: verify `https://vault.maktaba.home/alive`

## File map

| Path | Role |
| --- | --- |
| `~/git/projects/immich/docker-compose.yml` | Add `cloudflared` service + secret |
| `~/git/projects/immich/.gitignore` | Already ignores `secrets/` |
| `~/git/projects/immich/secrets/tunnel_token.txt` | Cloudflare tunnel token (host file, not git) |
| `/srv/immich/secrets/tunnel_token.txt` | Canonical secret location (symlink from project like `db_pw`) |
| `~/git/projects/immich/healthcheck.sh` | Assert `cloudflared` / `immich_cloudflared` running |
| `~/git/projects/immich/README.md` | Point at public URL + ops docs |
| `basehome-ops/docs/immich/operations.md` | Dual URLs, phone headers, friend runbook, 100MB note |
| `basehome-ops/docs/immich/public-access-runbook.md` | Friend allowlist + token rotation procedures |
| `basehome-ops/docs/infrastructure/topology.yaml` | Add public URL / tunnel edge |
| `basehome-ops/docs/TASKS.md` | Track public-access tasks |
| Cloudflare Zero Trust dashboard | Tunnel, DNS, Access app, policies, rate limit |

---

### Task 1: Cloudflare Tunnel + DNS for `photos.homz.fyi`

**Files:** none in git yet (dashboard + local secret file)

**Produces:** Working tunnel hostname that reaches Immich **without** Access (temporary; Access added in Task 3). Or with Access deferred until Task 3 — prefer bring tunnel up first, verify origin, then lock Access.

- [ ] **Step 1: Create Zero Trust tunnel**

In Cloudflare Dashboard → Zero Trust → Networks → Tunnels → Create tunnel (Cloudflared).

Name: `maktaba-immich`.

Copy the tunnel **token** once. Do not paste into chat logs or commit it.

- [ ] **Step 2: Store tunnel token on maktaba**

```bash
sudo mkdir -p /srv/immich/secrets
sudo install -m 600 /dev/null /srv/immich/secrets/tunnel_token.txt
# paste token into the file (no trailing newline required; echo -n preferred)
sudo nano /srv/immich/secrets/tunnel_token.txt
# or: printf '%s' 'TOKEN_HERE' | sudo tee /srv/immich/secrets/tunnel_token.txt >/dev/null
sudo chmod 600 /srv/immich/secrets/tunnel_token.txt
mkdir -p ~/git/projects/immich/secrets
ln -sfn /srv/immich/secrets/tunnel_token.txt ~/git/projects/immich/secrets/tunnel_token.txt
```

Save a copy of the token in Vaultwarden (secure note: `Cloudflare tunnel maktaba-immich`).

- [ ] **Step 3: Add public hostname in tunnel config (dashboard)**

Public hostname:

| Field | Value |
| --- | --- |
| Subdomain | `photos` |
| Domain | `homz.fyi` |
| Service | `http://immich_server:2283` |

(If the dashboard connector runs before Docker DNS exists, you may briefly use `http://172.x.x.x:2283` only for debugging; production must use Docker DNS name `immich_server` once cloudflared is on the Immich network — Task 2.)

Ensure DNS CNAME `photos.homz.fyi` is proxied (orange cloud) to the tunnel.

- [ ] **Step 4: Verify DNS exists**

```bash
dig +short photos.homz.fyi
# expect Cloudflare anycast IPs (proxied), not the home WAN IP
```

**Test:** DNS resolves via Cloudflare. Origin test completes in Task 2 after compose is up.

---

### Task 2: Add `cloudflared` to Immich compose

**Files:**
- Modify: `~/git/projects/immich/docker-compose.yml`
- Modify: `~/git/projects/immich/healthcheck.sh`
- Modify: `~/git/projects/immich/README.md` (brief pointer)

**Consumes:** `secrets/tunnel_token.txt` from Task 1  
**Produces:** Stable connector on Immich `default` network

- [ ] **Step 1: Add secret + service to compose**

Append to `docker-compose.yml` (verified against `cloudflared tunnel run --help`: `--token-file` / `$TUNNEL_TOKEN_FILE`):

```yaml
  cloudflared:
    container_name: immich_cloudflared
    image: cloudflare/cloudflared:latest
    restart: unless-stopped
    command:
      - tunnel
      - --no-autoupdate
      - run
      - --token-file
      - /run/secrets/tunnel_token
    secrets:
      - tunnel_token
    depends_on:
      - immich-server
    networks:
      - default
    mem_limit: 128M
```

Under existing `secrets:` add:

```yaml
  tunnel_token:
    file: ./secrets/tunnel_token.txt
```

- [ ] **Step 2: Bring up cloudflared**

```bash
cd ~/git/projects/immich
docker compose up -d cloudflared
docker compose ps
docker logs immich_cloudflared --tail 50
```

Expected: connector registered; no auth errors; tunnel status Healthy in dashboard.

- [ ] **Step 3: Origin smoke (before Access)**

From a network path that can hit Cloudflare (or temporarily with Access off):

```bash
curl -sS -o /dev/null -w '%{http_code}\n' https://photos.homz.fyi/api/server/ping
```

Expected: `200` (or Immich ping JSON). If Access is already on, expect `302`/`403` to Access login — then test origin via:

```bash
docker exec immich_cloudflared wget -qO- http://immich_server:2283/api/server/ping
```

Expected: Immich ping OK.

- [ ] **Step 4: Extend healthcheck**

In `healthcheck.sh`, after container list / memory section, also require `immich_cloudflared` running:

```bash
if docker inspect -f '{{.State.Running}}' immich_cloudflared 2>/dev/null | grep -q true; then
  echo "OK: immich_cloudflared is running"
else
  echo "ALERT: immich_cloudflared is not running"
  exit 1
fi
```

Update the `[n/m]` step labels. Keep vault `/alive` check.

```bash
./healthcheck.sh
```

Expected: passes including cloudflared + vault neighbor.

- [ ] **Step 5: Commit compose + healthcheck + README pointer** (immich repo)

```bash
cd ~/git/projects/immich
git add docker-compose.yml healthcheck.sh README.md
git status   # confirm secrets/ and .env NOT staged
git commit -m "$(cat <<'EOF'
Add cloudflared tunnel sidecar for photos.homz.fyi.

EOF
)"
```

---

### Task 3: Cloudflare Access (OTP) + household + guest policies + service token

**Files:** none required in git; document outcomes in Task 5

**Produces:** Public hostname deny-by-default except allowlisted emails and service token

- [ ] **Step 1: Enable One-time PIN IdP**

Zero Trust → Integrations → Identity providers → add **One-time PIN** if not already present.

- [ ] **Step 2: Create Access application**

- Type: Self-hosted  
- Application domain: `photos.homz.fyi` (all paths)  
- Identity providers: One-time PIN only  
- Session duration: choose something practical (e.g. 24h–7d)

- [ ] **Step 3: Policy — Household (Allow)**

Include emails: your email + each household member email.

Name: `immich-household`.

- [ ] **Step 4: Policy — Guests (Allow)**

Create a second Allow policy `immich-guests` with an email include list you will edit when sharing. Start empty or with a test email.

- [ ] **Step 5: Service token for mobile**

Zero Trust → Access → Service auth → Create service token (`immich-mobile`).

Save Client ID + Client Secret in Vaultwarden.

Add Access policy **Service Auth** / Bypass for that service token on the same application (order: service token usable by apps; humans still use OTP).

- [ ] **Step 6: Verify Access gate**

Incognito browser, no cookies:

```text
https://photos.homz.fyi
```

Expected: Cloudflare Access email prompt — **not** Immich login first.

Enter allowlisted email → receive OTP → after Access, Immich UI/login appears.

Enter non-allowlisted email → no code / access denied.

**Test:** Unallowlisted blocked; allowlisted reaches Immich.

---

### Task 4: Immich hardening + public URL settings

**Files:** Immich admin UI (no compose change required unless external URL is env-based)

- [ ] **Step 1: Confirm registration disabled**

Admin → Settings → ensure public registration / sign-up is **disabled**.

- [ ] **Step 2: Set public / external URL for shares**

Wherever Immich stores server external domain / OAuth redirect / share base URL, set:

`https://photos.homz.fyi`

so generated share links use the public host (not `photos.maktaba.home`).

- [ ] **Step 3: Create household Immich users** (if not already)

Admin → Users → create accounts (you remain sole admin).

- [ ] **Step 4: Sanity on private admin URL**

```bash
curl -k -sS -o /dev/null -w '%{http_code}\n' https://photos.maktaba.home/api/server/ping
curl -k -sS -o /dev/null -w '%{http_code}\n' https://vault.maktaba.home/alive
```

Expected: Immich ping OK; vault alive OK.

**Test:** Can log in as admin on Tailscale URL; registration closed on public path after Access.

---

### Task 5: Rate limiting + zone notes

**Files:** document rule ID/expression in operations.md (Task 6)

- [ ] **Step 1: Add rate limit for `photos.homz.fyi`**

Cloudflare zone `homz.fyi` → Security → WAF → Rate limiting rules.

Prefer host-scoped rule for `photos.homz.fyi` (Free plans often allow **1** rate-limit rule — do not burn it on unrelated hosts if this is the shared zone budget).

Suggested starting point (adjust to plan field limits):

- Match: hostname `photos.homz.fyi`
- Threshold: e.g. 100–300 requests / minute / IP (tighten later if abused)
- Action: Block

If Free plan only allows path-based expressions without host, use the single rule on Immich auth paths if available, and note the limitation in ops.

- [ ] **Step 2: Record the exact rule**

Write the final expression, threshold, and action into `operations.md` so it is not tribal knowledge.

**Test:** Rapid curl loop from a throwaway IP/path should eventually 429/block; normal browsing still works. Do not lock yourself out without Access still allowing recovery via Tailscale private URL.

---

### Task 6: Docs, topology, TASKS, runbooks

**Files:**
- Create: `basehome-ops/docs/immich/public-access-runbook.md`
- Modify: `basehome-ops/docs/immich/operations.md`
- Modify: `basehome-ops/docs/infrastructure/topology.yaml`
- Modify: `basehome-ops/docs/TASKS.md`
- Regenerate: topology.md / topology.html via script

- [ ] **Step 1: Write `public-access-runbook.md`**

Must include:

1. **Share with a friend:** add email to `immich-guests` Access policy → send Immich share link on `photos.homz.fyi` → friend OTP → when done, remove email.  
2. **Phone setup:** server URL `https://photos.homz.fyi`; Advanced → custom headers `CF-Access-Client-Id` / `CF-Access-Client-Secret` from Vaultwarden note.  
3. **Rotate service token:** create new token → update phones → revoke old → update Vaultwarden.  
4. **Large video note:** CF ~100MB/request; public backup may fail for huge videos; optional rescue via Tailscale private URL (not required for household setup).  
5. **Admin habit:** prefer `https://photos.maktaba.home` for admin UI.

- [ ] **Step 2: Update `operations.md`**

Add table rows for public URL; link runbook; healthcheck mentions cloudflared; do not remove private URL docs.

- [ ] **Step 3: Update topology**

In `topology.yaml`, for Immich service notes / URLs, add public `https://photos.homz.fyi` and note Cloudflare Tunnel + Access. Add edge entry if the schema has a place for external edges.

```bash
cd ~/git/projects/basehome-ops
python3 scripts/render-topology.py
```

- [ ] **Step 4: Update TASKS.md**

Under Immich, add tasks for tunnel/Access/docs with status as you complete them; mark CA trust task context (public URL reduces need for Immich mobile CA; vault CA may still matter for vault).

- [ ] **Step 5: Commit basehome-ops docs**

```bash
cd ~/git/projects/basehome-ops
git add docs/immich/ docs/infrastructure/topology.yaml docs/infrastructure/topology.md docs/infrastructure/topology.html docs/TASKS.md
git commit -m "$(cat <<'EOF'
Document Immich public access via photos.homz.fyi Tunnel and Access.

EOF
)"
```

---

### Task 7: End-to-end acceptance

**Produces:** Checked “setup and be done” list from the design

- [ ] **Step 1: Unallowlisted browser blocked**

Incognito → `https://photos.homz.fyi` → Access deny / no OTP for random email.

- [ ] **Step 2: Allowlisted OTP works**

Your email → OTP → Immich login/UI.

- [ ] **Step 3: Phone with service tokens**

Configure headers → login as household user → backup/sync a photo.

- [ ] **Step 4: Admin + neighbor**

Tailscale admin URL works; `./healthcheck.sh` passes (includes cloudflared + vault alive).

- [ ] **Step 5: Guest email cycle**

Add test friend email to `immich-guests` → they OTP in → open a share link → remove email → confirm blocked.

- [ ] **Step 6: Confirm Vaultwarden not on tunnel**

Tunnel public hostnames list contains **only** `photos.homz.fyi`.

---

## Execution handoff

Plan saved to `basehome-ops/docs/immich/2026-09-08-public-access-plan.md`.

**Two options:**

1. **Inline execution** — run Tasks 1–7 in this session with checkpoints  
2. **Pause** — you execute from the plan (Cloudflare dashboard steps need your login)

Which approach?
