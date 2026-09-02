# Network topology

```
Internet → ISP router (192.168.11.1)
              └─ LAN 192.168.11.0/24
                    ├─ Proxmox pve (192.168.11.123, TS 100.91.123.54)
                    │     └─ USB wdred 3.7T at /mnt/pve/wdred
                    │           ├─ NFS maktaba/media → maktaba VM
                    │           └─ NFS shared → maktaba VM
                    └─ Kubuntu maktaba VM (192.168.11.141, TS 100.68.38.53)
                          ├─ Docker: Jellyfin, TubeArchivist, Caddy, …
                          ├─ dnsmasq (*.maktaba.home → 100.68.38.53)
                          └─ Vaultwarden stack
```

## DNS

- **LAN:** Tailscale split DNS — nameserver `100.68.38.53` for domain `maktaba.home`
- **On maktaba:** dnsmasq binds LAN + Tailscale IP; all `*.maktaba.home` → `100.68.38.53`
- Router DHCP DNS to maktaba optional; Tailscale is primary path for owned devices

## Remote access

- **Default:** Tailscale mesh (no port-forward)
- **Not exposed:** Jellyfin, TubeArchivist, Vaultwarden directly to internet
- See [vaultwarden remote-access](../vaultwarden/remote-access.md) for VPN options

## Storage

- **VM disk (71G):** ES, redis, ta-cache, Jellyfin config, compose, secrets
- **USB via NFS:** `data/media` (~38G+), `/mnt/wdred/shared` for future VMs
- See [usb-wdred-runbook](../maktaba/usb-wdred-runbook.md)
