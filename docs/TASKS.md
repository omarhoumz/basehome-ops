# Tasks — home infrastructure

Cross-service open work. Service-specific detail in each project's handoff/ops docs.

## Maktaba

| # | Task | Status | Doc |
| --- | --- | --- | --- |
| 1 | NFS tighten to `192.168.11.141` only on Proxmox | ✅ done 2026-09-03 | [usb-wdred-runbook.md](./maktaba/usb-wdred-runbook.md) |
| 2 | Retry failed video `KFuVjmiCoSk` in TubeArchivist | ✅ already downloaded | [handoff.md](./maktaba/handoff.md) |
| 3 | Automated backup timer for `/srv/maktaba/scripts/backup.sh` | ✅ done 2026-09-03 | [operations.md](./maktaba/operations.md) |
| 4 | SoundCloud download pipeline test | ~~cancelled~~ | TA is YouTube-only; yt-dlp supports it but needs separate pipeline |
| 5 | Proxmox SMART / wdred health script | ✅ done 2026-09-03 | `/usr/local/bin/wdred-health.sh`; cron daily 06:00 → `/var/log/wdred-health.log` |

## Vaultwarden

| # | Task | Status | Doc |
| --- | --- | --- | --- |
| 1 | Trust Caddy local CA on client devices (vault) | 🔄 in progress — vault Tailscale HTTPS; Immich mobile prefers public URL (no Immich CA needed) | [operations.md](./vaultwarden/operations.md) |
| 2 | Backup restore drill | ✅ done 2026-09-03 | Restored `vw-backup-20260903_020001`; healthcheck passed |

## Immich

| # | Task | Status | Doc |
| --- | --- | --- | --- |
| 1 | Create project repo with memory caps (`~/git/projects/immich`) | ✅ done 2026-09-05 | [README.md](../../immich/README.md) · [setup-plan.md](./immich/setup-plan.md) |
| 2 | Prepare runtime directories (`/srv/immich`, library on wdred shared) | ✅ done 2026-09-06 | Library: `/mnt/wdred/shared/immich/library` |
| 3 | Start stack & verify memory caps (`./healthcheck.sh`) | ✅ done 2026-09-06 | All containers healthy; caps enforced |
| 4 | Wire Caddy `photos.maktaba.home` on Vaultwarden edge | ✅ done 2026-09-06 | vault `/alive` OK after reload |
| 5 | Schedule Postgres backup at 02:30, REGISTRY → live | ✅ done 2026-09-06 | Cron + smoke backup 18M OK |
| 6 | Optional: dedicated NFS export `wdred/immich/library` (move off shared) | ⬜ later | [setup-plan.md](./immich/setup-plan.md) |
| 7 | Cloudflare Tunnel `maktaba-immich` + DNS `photos.homz.fyi` | ⬜ | [public-access-plan.md](./immich/2026-09-08-public-access-plan.md) |
| 8 | `cloudflared` in Immich compose + healthcheck | ⬜ | [public-access-plan.md](./immich/2026-09-08-public-access-plan.md) |
| 9 | Access app + `immich-household` / `immich-guests` + `immich-mobile` service token | ⬜ | [public-access-plan.md](./immich/2026-09-08-public-access-plan.md) |
| 10 | Immich hardening (registration off, public external URL) | ⬜ | [public-access-plan.md](./immich/2026-09-08-public-access-plan.md) |
| 11 | Zone rate limit for `photos.homz.fyi` | ⬜ | [operations.md](./immich/operations.md) |
| 12 | Docs / topology / runbook for public access | ✅ done 2026-09-08 | [public-access-runbook.md](./immich/public-access-runbook.md) · [operations.md](./immich/operations.md) |
| 13 | End-to-end acceptance (OTP, phone token, guest cycle, vault not on tunnel) | ⬜ | [public-access-plan.md](./immich/2026-09-08-public-access-plan.md) |

## Homepage

| # | Task | Status | Doc |
| --- | --- | --- | --- |
| 1 | Create project repo (`~/git/projects/homepage`) + `/srv/homepage/config` | ⬜ | [setup-plan.md](./homepage/setup-plan.md) |
| 2 | Bring up stack on `vaultwarden_default` (no host `:3000`) | ⬜ | [setup-plan.md](./homepage/setup-plan.md) |
| 3 | Wire Caddy `home.maktaba.home`; validate vault `/alive` | ⬜ | [setup-plan.md](./homepage/setup-plan.md) |
| 4 | Seed services YAML for Jellyfin / TA / Immich / Vaultwarden | ⬜ | [setup-plan.md](./homepage/setup-plan.md) |
| 5 | REGISTRY → live + `docs/homepage/operations.md` | ⬜ | [REGISTRY.md](../projects/REGISTRY.md) |

## Forgejo

| # | Task | Status | Doc |
| --- | --- | --- | --- |
| 1 | Create project repo (`~/git/projects/forgejo`) + `/srv/forgejo/{data,postgres,backups}` | ⬜ | [setup-plan.md](./forgejo/setup-plan.md) |
| 2 | Bring up Forgejo + Postgres; create admin; registration off | ⬜ | [setup-plan.md](./forgejo/setup-plan.md) |
| 3 | Wire Caddy `git.maktaba.home`; validate vault `/alive` | ⬜ | [setup-plan.md](./forgejo/setup-plan.md) |
| 4 | Schedule DB+data backup at 03:00; push test repo over HTTPS | ⬜ | [setup-plan.md](./forgejo/setup-plan.md) |
| 5 | REGISTRY → live + `docs/forgejo/operations.md` | ⬜ | [REGISTRY.md](../projects/REGISTRY.md) |

## Done recently

| Date | Item |
| --- | --- |
| 2026-09-08 | Immich public-access docs — runbook, dual URLs, topology Tunnel + Access |
| 2026-09-07 | Forgejo planned — REGISTRY + setup-plan + topology (`git.maktaba.home`) |
| 2026-09-07 | Homepage planned — REGISTRY + setup-plan + topology (`home.maktaba.home`) |
| 2026-09-06 | Immich live — `https://photos.maktaba.home`, RAM caps, Caddy + vault neighbor OK |
| 2026-09-03 | Vaultwarden backup restore drill — integrity ok, restore + healthcheck passed |
| 2026-09-03 | Proxmox wdred health script — SMART + mount + NFS; cron daily 06:00 |
| 2026-09-03 | Automated backup timer — daily 02:00, keeps last 7, tested OK |
| 2026-09-03 | NFS tightened to `192.168.11.141` only; mounts + stack verified |
| 2026-09-02 | `basehome-ops` repo — docs mirror on GitHub |
| 2026-09-02 | USB WD Red migration — media on NFS, VM disk ~43% |
| 2026-09-02 | Disk alerts in maktaba `healthcheck.sh` (85% warn, 95% fail) |
| 2026-08-27 | TubeArchivist v0.5.12 |
