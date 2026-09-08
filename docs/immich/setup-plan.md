# Immich setup plan

| Field | Value |
| --- | --- |
| **Status** | `live` — 17 GiB VM; container RAM caps; library on `shared/immich` |
| **Last updated** | 2026-09-06 |
| **Topology** | [`docs/infrastructure/topology.md`](../infrastructure/topology.md) · [scan HTML](../infrastructure/topology.html) |

Google Photos replacement on the existing Kubuntu guest, matching **maktaba** (NFS for bulk bytes) and **Vaultwarden** (own `/srv` stack + Caddy `tls internal` + Tailscale-only).

---

## Resource budget & container limits (17 GiB VM)

Host now has **~17 GiB** RAM (~13 GiB available with media + Vaultwarden idle). Still use **hard cgroup memory caps** so Immich cannot starve Elasticsearch or Jellyfin.

### Current VM memory distribution (~17 GiB total)

| Consumer | Typical usage | Role / Notes |
| --- | --- | --- |
| `archivist-es` | ~1.6 GiB | Elasticsearch JVM for TubeArchivist. Sensitive to memory pressure. |
| `tubearchivist` | ~0.7 GiB | Django app & background workers. |
| `jellyfin` | ~0.4 GiB | Media server. Spikes higher during active transcode/scans. |
| Vaultwarden + Caddy | ~0.15 GiB | Extremely lightweight; critical service. |
| OS / Kernel / buffers | ~1.50–2.00 GiB | System cache, network buffers. |
| **Baseline reserved** | **~4.5–5.0 GiB** | Must stay protected from OOM kills. |
| **Available for Immich** | **~12+ GiB** | Comfortable headroom; caps still keep blast radius small. |

### Container memory caps (`mem_limit`)

Every Immich service gets an explicit cap in `docker-compose.yml` to prevent kernel OOM spills into Elasticsearch or Jellyfin:

```yaml
services:
  immich-server:
    # ...
    mem_limit: 1500M

  immich-machine-learning:
    # ...
    mem_limit: 2500M
    environment:
      - MACHINE_LEARNING_WORKERS=1

  immich-postgres:
    # ...
    mem_limit: 2000M

  immich-redis:
    # ...
    mem_limit: 256M
```

- **Sum of caps:** ~6.25 GiB (well within ~12 GiB free).
- **Behavior under load:** If ML or Postgres exceeds its cap, the kernel kills **only that container's cgroup**. Docker restarts it (`restart: unless-stopped`) without taking down the rest of the VM.
- **Optional later:** Raise caps or expand swap if ML jobs restart under large libraries.

---

## Mitigations baked into the plan

### 1. Cgroup memory fencing (Elasticsearch & Jellyfin protection)
- **Risk:** TubeArchivist (`archivist-es`) and Jellyfin are the heaviest memory consumers on this VM. An unconstrained ML job or DB query will trigger the Linux OOM killer against whichever process is largest (often Elasticsearch).
- **Mitigation:** Strict `mem_limit` on all Immich containers. Set `MACHINE_LEARNING_WORKERS=1` in the ML container environment so face/search processing runs single-threaded and avoids multi-worker memory multiplication.

### 2. Vaultwarden Caddy edge isolation
- **Risk:** Immich shares `:443` with Vaultwarden via `vaultwarden-caddy`. A bad Caddyfile edit or broken proxy config takes down your password manager.
- **Mitigation:**
  - Before reloading Caddy: validate syntax (`docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile`).
  - Immediately after reloading Caddy: automated verification ritual hitting Vaultwarden's health endpoint:
    ```bash
    curl --silent --show-error --fail --insecure https://vault.maktaba.home/alive
    ```
  - Connect Immich to Caddy via an isolated Docker network (`edge`), or proxy to `127.0.0.1:2283`.

### 3. NFS mount race guard (protect root SSD)
- **Risk:** If the Kubuntu VM boots and NFS fails to mount, Docker will silently create `/srv/immich/library` on the local 71G root SSD. Uploaded photos will fill the OS disk, triggering Elasticsearch read-only lockouts.
- **Mitigation:** Add a preflight assertion in `healthcheck.sh` and startup scripts:
  ```bash
  findmnt -T /srv/immich/library | grep -q "192.168.11.123:/mnt/pve/wdred/immich/library"
  ```
  Fail and refuse to start or ingest if the library path is not backed by the NFS mount.

### 4. Staggered nightly backups
- **Risk:** Maktaba and Vaultwarden run automated backups at `02:00`. Running Immich DB dumps at the same time causes simultaneous disk IO contention on the USB WD Red.
- **Mitigation:** Schedule Immich Postgres backups at **02:30**. Backup dumps cover Postgres only (local disk); photos live on wdred protected by Proxmox `wdred-health.sh`.

### 5. Migration-ready layout (easy split to dedicated VM later)
- **Risk:** If photo volume grows or ML needs expand, migrating from a shared VM can be messy if files are scattered.
- **Mitigation:**
  - Library lives on its own NFS export (`/mnt/pve/wdred/immich/library`) — moving to a new VM only requires exporting to the new VM's IP.
  - App files stay strictly self-contained in `/srv/immich` and `~/git/projects/immich`.
  - Splitting later only requires copying `/srv/immich/postgres`, updating the NFS client IP, and repointing DNS.

---

## Decisions

| # | Decision | Why |
| --- | --- | --- |
| 1 | **Same VM day one** (with RAM limits) | Reuse Tailscale, dnsmasq wildcard, Docker stack; defer new VM overhead until needed |
| 2 | **Own stack** under `/srv/immich` | Immich brings Postgres + Redis + ML; independent restart and compose lifecycle |
| 3 | **Library on dedicated NFS** `wdred/immich/library` | Bulk photos on 3.7T USB disk; clean separation from Jellyfin media |
| 4 | **Postgres on local SSD only** | Official Immich requirement: network DB causes corruption |
| 5 | **Tailscale-only day one** | Private, secure mobile backup; no open ports on ISP router |
| 6 | **HTTPS via existing `:443` Caddy** | Reuses local CA trusted by mobile clients; validated with `/alive` checks |
| 7 | **Hard container memory limits** | Protects Elasticsearch, Jellyfin, and Vaultwarden from memory spikes |

---

## Target shape

```
Proxmox pve (192.168.11.123)
  └─ /mnt/pve/wdred/
       ├─ pve/                 (host-only backups)
       ├─ maktaba/media/       (live NFS → Jellyfin/TA)
       ├─ shared/              (live NFS)
       └─ immich/library/      ← NEW NFS export

Kubuntu maktaba VM (192.168.11.141 / TS 100.68.38.53)
  ├─ /srv/maktaba/             (unchanged: Jellyfin, TA, ES)
  ├─ /srv/vaultwarden/         (unchanged: Vaultwarden, Caddy :443)
  └─ /srv/immich/              ← NEW
       ├─ .env                 (secrets, chmod 600)
       ├─ postgres/            (LOCAL VM disk, mem_limit: 2000M)
       ├─ model-cache/         (LOCAL VM disk, mem_limit: 2500M)
       ├─ backups/             (scheduled DB dumps at 02:30)
       └─ library → NFS        (UPLOAD_LOCATION, 3.7T wdred)

Clients (Tailscale) → https://photos.maktaba.home
                         → vaultwarden-caddy:443 → immich-server:2283
```

---

## Phased rollout

### Phase 0 — Host guardrails
1. Increase VM swapfile to 2 GiB (`fallocate -l 2G /swapfile.new ...`) to provide burst cushion.
2. Verify existing stack memory with `free -h` and `docker stats`.

### Phase 1 — Storage (Proxmox + Kubuntu)
1. On Proxmox: `mkdir -p /mnt/pve/wdred/immich/library && chown 1000:1000 /mnt/pve/wdred/immich/library`.
2. On Proxmox `/etc/exports`:
   ```exports
   /mnt/pve/wdred/immich/library  192.168.11.141(rw,sync,no_subtree_check,all_squash,anonuid=1000,anongid=1000)
   ```
3. Run `exportfs -ra`.
4. On Kubuntu `/etc/fstab`:
   ```fstab
   192.168.11.123:/mnt/pve/wdred/immich/library  /srv/immich/library  nfs  vers=4.1,_netdev,hard,noatime,nofail,x-systemd.automount,x-systemd.mount-timeout=30s  0  0
   ```
5. Mount and run write test as UID 1000.

### Phase 2 — App stack setup
1. Clone / copy official release into `~/git/projects/immich/`.
2. Configure `docker-compose.yml` with explicit `mem_limit` attributes on all 4 services.
3. Configure `MACHINE_LEARNING_WORKERS=1` in `.env`.
4. Prepare `/srv/immich/{postgres,model-cache,backups}` with permissions.
5. Launch stack (`docker compose up -d`) and verify memory bounds via `docker stats`.

### Phase 3 — Edge & TLS integration
1. Add `photos.maktaba.home` to Vaultwarden `Caddyfile`.
2. Validate Caddy config before reload.
3. Reload Caddy and immediately test `https://vault.maktaba.home/alive` (must return 200).
4. Verify `https://photos.maktaba.home` responds over Tailscale.
5. Complete web admin account creation.

### Phase 4 — Ops hygiene & mobile test
1. Implement `healthcheck.sh` with mount verification (`findmnt`) and HTTPS checks.
2. Implement daily Postgres backup at `02:30`.
3. Test photo backup from mobile app with a small album.
4. Update `basehome-ops/projects/REGISTRY.md` from `planned` → `live`.
