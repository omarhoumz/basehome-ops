# Registry — home services

**Gate:** no new public-facing service without a row. Update the same day something goes live or changes IP/URL.

| Service | Host | LAN IP | Tailscale | URL(s) | Live path | Status | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| maktaba (media) | Kubuntu VM | 192.168.11.141 | 100.68.38.53 | player/yt.maktaba.home | `/srv/maktaba` | live | Jellyfin + TubeArchivist + Caddy; media on USB NFS |
| vaultwarden | Kubuntu VM | 192.168.11.141 | 100.68.38.53 | vault.maktaba.home | `/srv/vaultwarden` | live | Caddy TLS internal CA |
| dnsmasq | maktaba VM | 192.168.11.141 | 100.68.38.53 | `*.maktaba.home` | `/etc/dnsmasq.d/maktaba.conf` | live | Tailscale split DNS nameserver |
| proxmox | pve | 192.168.11.123 | 100.91.123.54 | — | — | live | USB `wdred` 3.7T, NFS server |
| wdred USB | pve | — | — | — | `/mnt/pve/wdred` | live | maktaba/media + shared NFS; pve/ for backups |

**Status values:** `planned` · `live` · `retired`
