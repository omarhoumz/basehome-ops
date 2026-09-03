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
| 1 | Trust Caddy local CA on all client devices | ~~cancelled~~ — Tailscale-only for now; revisit when public | [operations.md](./vaultwarden/operations.md) |
| 2 | Backup restore drill | ✅ done 2026-09-03 | Restored `vw-backup-20260903_020001`; healthcheck passed |

## Done recently

| Date | Item |
| --- | --- |
| 2026-09-03 | Vaultwarden backup restore drill — integrity ok, restore + healthcheck passed |
| 2026-09-03 | Proxmox wdred health script — SMART + mount + NFS; cron daily 06:00 |
| 2026-09-03 | Automated backup timer — daily 02:00, keeps last 7, tested OK |
| 2026-09-03 | NFS tightened to `192.168.11.141` only; mounts + stack verified |
| 2026-09-02 | `basehome-ops` repo — docs mirror on GitHub |
| 2026-09-02 | USB WD Red migration — media on NFS, VM disk ~43% |
| 2026-09-02 | Disk alerts in maktaba `healthcheck.sh` (85% warn, 95% fail) |
| 2026-08-27 | TubeArchivist v0.5.12 |
