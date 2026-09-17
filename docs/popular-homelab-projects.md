# Popular homelab projects (2025–2026)

Community-ranked shortlist for what to run next. Primary ranking: **2025 Self-Hosted Survey MVP votes** ([deployn.de](https://selfhosted-survey-2025.deployn.de/mvp/)), cross-checked with [selfh.st 2025](https://selfh.st/survey/2025-results/) (~4k responses) and recent stack guides.

**Not a deploy plan** — see [`REGISTRY.md`](../projects/REGISTRY.md) for what is live. Status here is basehome-relative.

| Status | Meaning |
| --- | --- |
| live | Running on basehome (see registry) |
| planned | Deploy plan exists; not live yet |
| candidate | Worth considering |
| skip / watch | Proprietary, overlap, or plumbing — note only |

## Top 10

| # | Project | MVP votes | Status | Explainer |
| --- | --- | --- | --- | --- |
| 1 | [Jellyfin](https://jellyfin.org) | 188 | live (maktaba) | Open-source media server for movies, TV, and music. Most-cited “can’t live without” app in 2025 surveys; Plex alternative with no paywall. |
| 2 | [Immich](https://immich.app) | 156 | live | Google Photos replacement. Live at `photos.maktaba.home` with cgroup RAM caps; see [operations](./immich/operations.md). |
| 3 | [Home Assistant](https://www.home-assistant.io) | 122 | candidate | Local smart-home hub for lights, sensors, Zigbee/Z-Wave/Matter. Best as HAOS VM on Proxmox when you want full add-ons. |
| 4 | [Vaultwarden](https://github.com/dani-garcia/vaultwarden) | 92 | live | Lightweight Bitwarden-compatible password server. Often the first self-host win; browser/mobile clients already exist. |
| 5 | [Plex](https://www.plex.tv) | 78 | skip / watch | Proprietary media server still widely used. Strong clients/remote features; overlap with Jellyfin already live — only if a client gap appears. |
| 6 | [Paperless-ngx](https://docs.paperless-ngx.com) | 60 | candidate | Scan → OCR → searchable document archive. Replaces “paper in Drive” for bills, IDs, and household paperwork. |
| 7 | [Nextcloud](https://nextcloud.com) | 57 | candidate | Personal cloud: files, sync, calendar, contacts, apps. Google Workspace / Dropbox–style suite; heavier than Syncthing or Immich alone. |
| 8 | [Sonarr](https://sonarr.tv) | 51 | candidate | TV library automation (*arr stack). Usually paired with Radarr, Prowlarr, and a download client; feeds Jellyfin libraries. |
| 9 | [Pi-hole](https://pi-hole.net) | 51 | candidate | Network-wide DNS sinkhole for ads/trackers. Alternative: AdGuard Home (cleaner UI, DoH/DoT). Overlaps with dnsmasq role on maktaba — plan carefully. |
| 10 | [Audiobookshelf](https://www.audiobookshelf.org) | 45 | candidate | Dedicated audiobook and podcast server with good mobile clients. Complements Jellyfin when audio libraries deserve their own UX. |

## Platform (not an app, but #1 base)

| Project | Survey signal | Status | Explainer |
| --- | --- | --- | --- |
| [Proxmox VE](https://www.proxmox.com) | Top host OS (~42%) | live (pve) | Type-1 hypervisor for VMs + LXC. Survey #1 home host; Docker Compose (~88%) usually runs inside guests. |

## Honorable mentions

| Project | Role | Explainer |
| --- | --- | --- |
| [Radarr](https://radarr.video) | Media automation | Movie half of the *arr stack; pairs with Sonarr + Prowlarr. |
| [AdGuard Home](https://adguard.com/adguard-home/overview.html) | DNS filtering | Pi-hole alternative with modern UI and built-in DoH/DoT. |
| [Nginx Proxy Manager](https://nginxproxymanager.com) | Reverse proxy | Survey-leading easy reverse proxy + TLS UI (~36% of reverse-proxy users). |
| [Traefik](https://traefik.io) | Reverse proxy | Label-driven proxy popular with Docker Compose; more “config as code” than NPM. |
| [Portainer](https://www.portainer.io) / [Dockge](https://github.com/louislam/dockge) | Container UI | Web UI for Docker; Dockge is compose-file–first and lighter. |
| [Uptime Kuma](https://uptime.kuma.pet) | Monitoring | Simple uptime checks and alerts for every service URL. |
| [Homepage](https://gethomepage.dev) | Start page / dashboard | YAML dashboard with service links. **live** at `home.maktaba.home` — see [operations](./homepage/operations.md). |
| [Forgejo](https://forgejo.org) | Git forge | Lightweight self-hosted GitHub alternative (Gitea fork). **planned** for basehome — see [setup-plan](./forgejo/setup-plan.md). |
| [Syncthing](https://syncthing.net) | File sync | Peer-to-peer folder sync without a central cloud; lighter than Nextcloud for “just sync these dirs.” |
| [Authentik](https://goauthentik.io) | SSO / IdP | Single sign-on in front of many apps once the service count grows. |
| *arr + [Jellyseerr](https://github.com/Fallenbagel/jellyseerr) | Media requests | Request UI for friends/family; Jellyseerr is the Jellyfin-oriented Overseerr fork. |
| [Tailscale](https://tailscale.com) / [WireGuard](https://www.wireguard.com) | Remote access | Mesh/VPN for private access without opening ports; Tailscale split DNS already used for `maktaba.home`. |
| [Open WebUI](https://openwebui.com) + [Ollama](https://ollama.com) | Local AI | Chat UI + local LLM runtime; needs GPU/VRAM for larger models. |

## Sources

- [Self-Hosted Survey 2025 — MVPs](https://selfhosted-survey-2025.deployn.de/mvp/)
- [selfh.st 2025 survey results](https://selfh.st/survey/2025-results/)
- [OSSAlt: Homelab software stack 2026](https://ossalt.com/guides/homelab-software-stack-guide-2026)
- [NetBird: 10 self-hosted apps 2026](https://netbird.io/knowledge-hub/10-self-hosted-apps-2026)
