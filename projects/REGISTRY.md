# Registry — home services

**Gate:** no new public-facing service without a row. Update the same day something goes live or changes IP/URL.

| Service | Host | LAN IP | Tailscale | URL(s) | Live path | Status | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| maktaba (media) | Kubuntu VM | 192.168.11.141 | 100.68.38.53 | player/yt.maktaba.home | `/srv/maktaba` | live | Jellyfin + TubeArchivist + Caddy; media on USB NFS |
| vaultwarden | Kubuntu VM | 192.168.11.141 | 100.68.38.53 | vault.maktaba.home | `/srv/vaultwarden` | live | Caddy TLS internal CA |
| immich | Kubuntu VM | 192.168.11.141 | 100.68.38.53 | photos.maktaba.home | `/srv/immich` | live | Memory capped (~6.25G); library on `shared/immich`; see [operations](../docs/immich/operations.md) |
| homepage | Kubuntu VM | 192.168.11.141 | 100.68.38.53 | home.maktaba.home | `/srv/homepage` | live | gethomepage; Docker status via socket RO; Caddy via vault edge; see [operations](../docs/homepage/operations.md) |
| forgejo | Kubuntu VM | 192.168.11.141 | 100.68.38.53 | git.maktaba.home | `/srv/forgejo` | planned | Self-hosted Git forge; Postgres on VM SSD; see [setup-plan](../docs/forgejo/setup-plan.md) |
| dnsmasq | maktaba VM | 192.168.11.141 | 100.68.38.53 | `*.maktaba.home` | `/etc/dnsmasq.d/maktaba.conf` | live | Tailscale split DNS nameserver |
| proxmox | pve | 192.168.11.123 | 100.91.123.54 | — | — | live | USB `wdred` 3.7T, NFS server |
| wdred USB | pve | — | — | — | `/mnt/pve/wdred` | live | maktaba/media + shared NFS (incl. immich/library for now); pve/ for backups |

**Status values:** `planned` · `live` · `retired`
