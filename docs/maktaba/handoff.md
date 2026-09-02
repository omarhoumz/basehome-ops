# Handoff: Maktaba media server

| Field | Value |
| --- | --- |
| **Status** | `active` |
| **Last updated** | 2026-09-02 |
| **Next focus** | Routine ops — NFS tighten on Proxmox, optional follow-ups |

> **For whoever picks this up:** Read this file first, then run `./scripts/healthcheck.sh` on the server. **Update status, date, and open items as you work.**

---

## What this is

Home media stack on Proxmox **Kubuntu VM** `maktaba` (`192.168.11.141`, Tailscale `100.68.38.53`).

| URL | Service |
| --- | --- |
| http://player.maktaba.home | Jellyfin |
| http://yt.maktaba.home | TubeArchivist |

**Live stack path:** `/srv/maktaba`

**Proxmox host:** `pve` at `192.168.11.123` (Tailscale `100.91.123.54`).

**Docs mirror:** this repo under `docs/maktaba/`.

---

## Verified state (2026-09-02)

- All containers healthy; TubeArchivist **v0.5.12**
- Media on USB NFS: `192.168.11.123:/mnt/pve/wdred/maktaba/media` → `/srv/maktaba/data/media`
- Shared NFS: `/mnt/wdred/shared`
- VM root disk **~43%** used
- Tailscale split DNS for `maktaba.home`; exit node **off**
- YouTube via browser cookies in Redis

---

## Open items

See [`docs/TASKS.md`](../TASKS.md).

---

## Quick start (on server)

```bash
cd /srv/maktaba
./scripts/healthcheck.sh
findmnt /srv/maktaba/data/media /mnt/wdred/shared
df -h / /srv/maktaba/data/media
```

---

## Completion

- **2026-09-02:** USB WD Red migration complete — media on NFS (`3.7T`), stack healthy.
