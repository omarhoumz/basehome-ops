# USB WD Red runbook — Proxmox → Kubuntu → maktaba

| Field | Value |
| --- | --- |
| **Status** | `completed` — migration done 2026-09-02 |
| **Topology** | Proxmox host → **Kubuntu VM** → Docker (maktaba stack) |
| **Disk** | ~3.7 TB USB WD40EFZZ on Proxmox (`/mnt/pve/wdred`) |
| **Related** | [usb-storage-plan.html](./usb-storage-plan.html), draft PR [omarhoumz#1](https://github.com/omarhoumz/omarhoumz/pull/1) |

## Decisions (grilled)

| # | Decision |
| --- | --- |
| 1 | NFS from **Proxmox host** (not OMV VM) |
| 2 | Day-one folders: `pve/`, `maktaba/media/`, `shared/`; **migrate media in same plan** |
| 3 | Proxmox Directory `wdred`: `backup,iso,vztmpl` only — **never** `images`/`rootdir` |
| 4 | NFS exports **`maktaba/media` + `shared` only**; `pve/` stays host-local |
| 5 | NFS open to **LAN subnet** — document [tighten to guest IPs later](#follow-ups) |
| 6 | NFS squash to **UID/GID 1000** (matches maktaba `basehome` / TA `HOST_UID`) |
| 7 | Host mount **`/mnt/pve/wdred`**, storage ID **`wdred`** |
| 8 | Kubuntu mounts NFS **in place** at `/srv/maktaba/data/media`; **Kubuntu runs rsync** |
| 9 | Proxmox: **`nofail` + automount + `is_mountpoint yes`** |
| 10 | Kubuntu also mounts **`/mnt/wdred/shared`** day one |
| 11 | **SMART + udev + health script** on format day |
| 12 | Kubuntu: **`nofail` + automount**; stop stack before unplug |
| 13 | **Hard gate** before wipe (identity checkpoint) |
| 14 | Delete **`data/media.bak`** manually after verification checklist |

## Architecture

```
USB disk
  └─ Proxmox host: /mnt/pve/wdred/
       ├─ pve/              → pvesm (backup, iso, vztmpl) — NOT exported
       ├─ maktaba/media/    → NFS export → Kubuntu:/srv/maktaba/data/media → Docker
       └─ shared/           → NFS export → Kubuntu:/mnt/wdred/shared

Kubuntu local (unchanged):
  /srv/maktaba/data/es, redis, ta-cache, jellyfin/ …  ← stay on VM disk
```

**Core rule:** Only Proxmox mounts the USB block device. All guests use NFS.

---

## Variables (fill before you start)

| Variable | Example | How to find |
| --- | --- | --- |
| `PROXMOX_LAN_IP` | `192.168.11.123` | Proxmox host `pve` |
| `KUBUNTU_LAN_IP` | `192.168.11.141` | This VM (`maktaba` hostname) |
| `LAN_SUBNET` | `192.168.11.0/24` | Your LAN |
| `DISK_BY_ID` | `/dev/disk/by-id/ata-WDC_WD40EFZZ-…` | Proxmox: `ls -l /dev/disk/by-id/ \| grep -v part` |
| `USB_VID_PID` | from `lsusb` | Proxmox: `lsusb` (for udev rule) |

---

## Phase 0 — Inventory (Proxmox shell)

Run on **`pve`**, not Kubuntu.

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,TRAN,TYPE,FSTYPE,MOUNTPOINTS
ls -l /dev/disk/by-id/ | grep -v part
lsusb
pveversion
pvesm status
cat /etc/fstab
grep -r . /etc/exports 2>/dev/null || true
```

### CHECKPOINT 0 — Confirm disk identity

Paste output and verify:

- [ ] Exactly **one** ~3.7T disk is the USB WD (model matches label).
- [ ] Proxmox **OS disk** is a **different** device (usually smaller NVMe/SSD).
- [ ] You have the **whole-disk** `by-id` path (no `-part` suffix).
- [ ] SMART works: `smartctl -i -H -d sat "$DISK_BY_ID"` → **PASSED**, model **WD40EFZZ** (or your SKU).

**Stop.** Do not format until you explicitly confirm: **“Yes, wipe THIS disk: $DISK_BY_ID”**.

---

## Phase 1 — Format + host mount (Proxmox)

All commands on **Proxmox**. Replace `DISK_BY_ID`.

```bash
DISK_BY_ID=/dev/disk/by-id/ata-WDC_WD40EFZZ-CHANGE-ME

wipefs -a "$DISK_BY_ID"
sgdisk -Z "$DISK_BY_ID"
sgdisk -n 1:1MiB:0 -t 1:8300 -c 1:wdred "$DISK_BY_ID"
partprobe "$DISK_BY_ID"
udevadm settle
test -b "${DISK_BY_ID}-part1"

mkfs.ext4 -L wdred -m 0 -T largefile4 \
  -E lazy_itable_init=0,lazy_journal_init=0,nodiscard \
  "${DISK_BY_ID}-part1"
```

Prepare mountpoint (Path B safety):

```bash
mkdir -p /mnt/pve/wdred
chattr +i /mnt/pve/wdred
blkid "${DISK_BY_ID}-part1"
```

Add to **`/etc/fstab`** on Proxmox (use UUID from `blkid`):

```fstab
UUID=YOUR-UUID-HERE  /mnt/pve/wdred  ext4  noatime,errors=remount-ro,nofail,x-systemd.device-timeout=30s,x-systemd.automount,x-systemd.mount-timeout=30s  0  0
```

Mount and verify:

```bash
systemctl daemon-reload
mount /mnt/pve/wdred
findmnt /mnt/pve/wdred
df -h /mnt/pve/wdred
chattr -i /mnt/pve/wdred
```

### CHECKPOINT 1 — Host mount

- [ ] `findmnt /mnt/pve/wdred` shows the WD UUID, **not** root filesystem.
- [ ] `df -h` shows ~3.5T available on `/mnt/pve/wdred`.

---

## Phase 2 — Folder layout + Proxmox storage (Proxmox)

```bash
mkdir -p /mnt/pve/wdred/{pve,maktaba/media,shared}
chown -R 1000:1000 /mnt/pve/wdred/maktaba /mnt/pve/wdred/shared

pvesm add dir wdred --path /mnt/pve/wdred/pve \
  --content backup,iso,vztmpl \
  --prune-backups 'keep-last=3'
pvesm status | grep wdred
```

> **Note:** `--is-mountpoint yes` fails when the storage path is a subfolder (`/mnt/pve/wdred/pve`) rather than the mount root. Omit it; the host fstab still guards the USB mount.

### CHECKPOINT 2

- [ ] `wdred` active; content is `backup,iso,vztmpl` only — **no** `images`/`rootdir`.

---

## Phase 3 — NFS server (Proxmox)

```bash
apt install -y nfs-kernel-server
```

**`/etc/exports`** — restrict to maktaba VM only (`192.168.11.141`):

```exports
/mnt/pve/wdred/maktaba/media  192.168.11.141(rw,sync,no_subtree_check,all_squash,anonuid=1000,anongid=1000)
/mnt/pve/wdred/shared           192.168.11.141(rw,sync,no_subtree_check,all_squash,anonuid=1000,anongid=1000)
```

Do **not** use `192.168.11.0/24` unless you intentionally want every LAN host to mount the share.

```bash
exportfs -ra
systemctl enable --now nfs-server
showmount -e localhost
```

### CHECKPOINT 3 — From Kubuntu

```bash
showmount -e PROXMOX_LAN_IP
sudo mount -t nfs PROXMOX_LAN_IP:/mnt/pve/wdred/maktaba/media /tmp/nfs-test
touch /tmp/nfs-test/.write-test && rm /tmp/nfs-test/.write-test
sudo umount /tmp/nfs-test
```

- [ ] Write test OK; **`pve/` not exported**.

---

## Phase 4 — Host monitoring (Proxmox)

- `smartd` with `-d sat` on whole-disk by-id
- udev rule: USB autosuspend off (VID:PID from `lsusb`)
- Health script (see draft PR `wdred-health.sh`)

### CHECKPOINT 4

- [ ] SMART PASSED; `smartd` active; health script exits 0

---

## Phase 5 — Kubuntu persistent mounts

```bash
sudo apt install -y nfs-common
sudo mkdir -p /srv/maktaba/data/media /mnt/wdred/shared
```

**`/etc/fstab`** on Kubuntu:

```fstab
PROXMOX_LAN_IP:/mnt/pve/wdred/maktaba/media  /srv/maktaba/data/media  nfs  vers=4.1,_netdev,hard,noatime,nofail,x-systemd.automount,x-systemd.mount-timeout=30s  0  0
PROXMOX_LAN_IP:/mnt/pve/wdred/shared           /mnt/wdred/shared        nfs  vers=4.1,_netdev,hard,noatime,nofail,x-systemd.automount,x-systemd.mount-timeout=30s  0  0
```

Do **not** mount at final media path until Phase 6 cutover if local `data/media` still has files.

---

## Phase 6 — Media migration (Kubuntu)

**Kubuntu copies.** Proxmox only provides empty NFS destination.

```bash
cd /srv/maktaba
docker compose stop tubearchivist jellyfin

sudo mkdir -p /mnt/wdred-import
sudo mount -t nfs PROXMOX_LAN_IP:/mnt/pve/wdred/maktaba/media /mnt/wdred-import
sudo rsync -aH --info=progress2 data/media/ /mnt/wdred-import/
du -sh data/media /mnt/wdred-import
```

### CHECKPOINT 6a

- [ ] Sizes match; spot-check channel folders.

**Cutover:**

```bash
sudo umount /mnt/wdred-import
sudo mv data/media data/media.bak
sudo mkdir data/media
sudo systemctl daemon-reload
sudo mount /srv/maktaba/data/media
sudo mount /mnt/wdred/shared
findmnt /srv/maktaba/data/media

docker compose up -d
./scripts/healthcheck.sh
```

### CHECKPOINT 6c

- [ ] UIs healthy; playback OK; test download OK; `df -h /` improved on Kubuntu.

---

## Phase 7 — Delete old copy (manual)

When satisfied:

```bash
sudo rm -rf /srv/maktaba/data/media.bak
```

Checklist: playback, new download, Jellyfin scan, comfortable root disk free space.

---

## Runbooks

### Safe unplug

**Kubuntu:** `docker compose stop tubearchivist jellyfin` → `umount` media + shared.

**Proxmox:** `umount /mnt/pve/wdred` → unplug 12V/USB.

### Remount

12V first → USB → Proxmox `mount /mnt/pve/wdred` → Kubuntu mount fstab paths → `docker compose up -d`.

---

## Follow-ups

| Item | Notes |
| --- | --- |
| Tighten NFS to guest IPs | Use `192.168.11.141` only in `/etc/exports` (not whole LAN) — **pending apply on Proxmox** |
| ~~Delete `media.bak`~~ | Done 2026-09-02 |
| NAS migration | Future RAID will **wipe** ext4 — copy off first |

## What stays local on Kubuntu

Only **`data/media`** moves to NFS. Keep **`data/es`**, **redis**, **ta-cache**, **jellyfin config** on VM disk.
