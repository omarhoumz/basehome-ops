# Homepage service API widgets — implementation plan

> **For agentic workers:** Implement task-by-task. Steps use checkbox (`- [ ]`) syntax. Do not commit secrets. Prefer Docker DNS URLs over public/Tailscale hostnames where possible.

**Goal:** Add live service stats on Homepage tiles for Jellyfin, Immich, and TubeArchivist using Homepage service widgets + `HOMEPAGE_VAR_*` secrets.

**Architecture:** Keys live only in `/srv/homepage/.env` (chmod 600), referenced from Compose `env_file` and interpolated in `services.yaml` as `{{HOMEPAGE_VAR_*}}`. Widgets call each app over the Docker network (or existing private URLs). Vaultwarden has no first-class Homepage widget — skip unless a Custom API is worth it later.

**Tech stack:** gethomepage service widgets, Docker Compose env files, existing maktaba / Immich / Vaultwarden stacks.

**Spec / ops:** [operations.md](./operations.md) · [setup-plan.md](./setup-plan.md)

## Global constraints

- No API keys in git (`homepage` or `basehome-ops`)
- `.env` only under `/srv/homepage/.env` (or project `.env` gitignored) — chmod 600
- After any Caddy change: `curl -k -sf https://vault.maktaba.home/alive`
- Prefer URLs Homepage can reach: Docker DNS (`http://jellyfin:8096`, `http://immich_server:2283`, `http://tubearchivist:8000`) when on shared networks; otherwise Tailscale HTTPS URLs with `-k` not available inside widget — so Docker network joins may be required
- Homepage currently only joins `vaultwarden_default` — Immich is already on that network; Jellyfin/TA are on maktaba’s network — **must join or use host-published URLs**

## Network decision (do first)

Homepage must reach each API. Options:

| Option | Approach | Trade-off |
| --- | --- | --- |
| **A (recommended)** | Attach Homepage to `maktaba` compose network as external + keep `vaultwarden_default` | Clean Docker DNS; no TLS/CA issues |
| B | Widget URLs = `http://192.168.11.141:…` host ports if published | Brittle; only if ports exist on host |
| C | Widget URLs = `https://….maktaba.home` | Needs trusted CA inside Homepage container — avoid |

**Plan assumes Option A** unless host ports already work.

---

### Task 1: Confirm reachability + networks

- [ ] **Step 1: List networks**

```bash
docker inspect jellyfin --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool
docker inspect tubearchivist --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool
docker inspect immich_server --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool
docker network ls
```

- [ ] **Step 2: Choose network name** for maktaba stack (likely `maktaba_default` or similar) and note Immich is already on `vaultwarden_default`.

- [ ] **Step 3: Document chosen URLs** in this file’s “URL map” section below once verified.

**URL map (fill during Task 1):**

| Service | Widget `url` | Notes |
| --- | --- | --- |
| Jellyfin | _TBD_ | Create API key in Dashboard → Advanced → API Keys |
| Immich | _TBD_ | API key with `server.statistics` (or full access if scoped key fails); widget `version: 2` if Immich ≥ 1.118 |
| TubeArchivist | _TBD_ | API token from TA settings → Application |

---

### Task 2: Secrets layout

- [ ] **Step 1: Create `/srv/homepage/.env`** (not in git)

```bash
touch /srv/homepage/.env
chmod 600 /srv/homepage/.env
```

Contents (placeholders):

```bash
HOMEPAGE_VAR_JELLYFIN_KEY=
HOMEPAGE_VAR_IMMICH_KEY=
HOMEPAGE_VAR_TA_KEY=
```

- [ ] **Step 2: Wire Compose**

In `~/git/projects/homepage/docker-compose.yml` under `homepage:`:

```yaml
    env_file:
      - /srv/homepage/.env
```

- [ ] **Step 3: Symlink or document** — keep `~/git/projects/homepage/.env.example` listing the three `HOMEPAGE_VAR_*` names with empty values (already gitignored `.env`).

- [ ] **Step 4: Recreate homepage** so env loads: `docker compose up -d`

---

### Task 3: Jellyfin widget

- [ ] **Step 1: Create API key** in Jellyfin admin → Advanced → API Keys; store in `HOMEPAGE_VAR_JELLYFIN_KEY`.

- [ ] **Step 2: Edit `services.yaml`** Jellyfin entry:

```yaml
    - Jellyfin:
        href: http://player.maktaba.home
        description: Movies / TV
        icon: jellyfin.png
        server: local
        container: jellyfin
        widget:
          type: jellyfin
          url: http://jellyfin:8096   # or verified URL from Task 1
          key: "{{HOMEPAGE_VAR_JELLYFIN_KEY}}"
          enableNowPlaying: true
          # version: 2   # if Jellyfin >= 10.12
```

- [ ] **Step 3: Sync + verify** — tile shows now-playing / library fields without API error; vault `/alive` unchanged.

```bash
rsync -a ~/git/projects/homepage/config/ /srv/homepage/config/
curl -k -sf https://vault.maktaba.home/alive
```

---

### Task 4: Immich widget

- [ ] **Step 1: Create Immich API key** (Account Settings → API Keys) with `server.statistics` (escalate to broader scopes only if Homepage returns API errors).

- [ ] **Step 2: Add widget** under Immich in `services.yaml`:

```yaml
        widget:
          type: immich
          url: http://immich_server:2283
          key: "{{HOMEPAGE_VAR_IMMICH_KEY}}"
          version: 2
```

- [ ] **Step 3: Verify** fields users/photos/videos/storage appear; private admin URL still works.

---

### Task 5: TubeArchivist widget

- [ ] **Step 1: Copy API token** from TA Settings → Application.

- [ ] **Step 2: Add widget**:

```yaml
        widget:
          type: tubearchivist
          url: http://tubearchivist:8000
          key: "{{HOMEPAGE_VAR_TA_KEY}}"
          fields: ["downloads", "videos", "channels", "playlists"]
```

- [ ] **Step 3: Verify** no API error on tile (if downloads field flakes, drop it from `fields`).

---

### Task 6: Docs + TASKS

- [ ] **Step 1: Update** `docs/homepage/operations.md` — secrets path, which widgets are live, network attachments.
- [ ] **Step 2: Mark** TASKS Homepage item 9 done with date.
- [ ] **Step 3: Commit** `homepage` (compose, services.yaml, .env.example, healthcheck) and `basehome-ops` docs — **no** `.env`.

---

### Out of scope

- Vaultwarden service widget (no official Homepage widget)
- Forgejo widget until Forgejo is live
- Glances / Uptime Kuma
- Publishing docker-socket-proxy to the host

---

### Acceptance

- [ ] Homepage has **no** `/var/run/docker.sock` mount (proxy only) — already done separately
- [ ] Jellyfin / Immich / TA tiles show live API stats without errors
- [ ] `git status` in homepage/basehome-ops shows no secret files
- [ ] `https://vault.maktaba.home/alive` OK
