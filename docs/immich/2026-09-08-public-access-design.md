# Immich public access — design

**Date:** 2026-09-08  
**Status:** approved (grill + design §§1–3)  
**Plan:** [2026-09-08-public-access-plan.md](./2026-09-08-public-access-plan.md)

## Goal

Household phones use Immich **without Tailscale or local CA**. Friends can be granted temporary access by **email**. Admin is locked down. Vaultwarden stays private for now, with a clean path for a later separate publish.

## Decisions (grill)

| Topic | Decision |
| --- | --- |
| Public edge | Cloudflare Tunnel only (no port-forward) |
| Public hostname | `https://photos.homz.fyi` (zone `homz.fyi` on Cloudflare) |
| Private admin URL | `https://photos.maktaba.home` (Tailscale / existing vaultwarden-caddy) |
| Access | Cloudflare Access on **all paths**; One-time PIN (email) |
| Allowlists | Permanent: you + household emails; Guests: add/remove friend emails when sharing |
| Mobile | Immich accounts + Access **service-token** headers on phones |
| Friends | Add email to Access → OTP → Immich; runbook for add/remove |
| Registration | Off — only admin creates users |
| Rate limits | From day one (Cloudflare zone budget) |
| Ops | Backup/restore/update/monitor notes from day one |
| Vaultwarden | Private now; later = **own** hostname + Access + connector — never Immich tunnel |
| Large uploads | CF ~100MB/request limit **accepted**; household stays public-URL-only |
| Connector | `cloudflared` on maktaba in Immich compose → `immich_server:2283` |

## Topology

```
Internet
  → photos.homz.fyi
       Cloudflare: Access (OTP) + rate limits + service-token bypass for apps
       → cloudflared (Immich compose, maktaba)
            → immich_server:2283   [Immich ONLY]

Tailscale / LAN
  → photos.maktaba.home (vaultwarden-caddy, tls internal)
       → immich_server:2283       [admin UI habit]

Vaultwarden: Tailscale-only (unchanged).
```

## Why Access on all paths (not only `/admin`)

Immich admin UI lives under `/admin/*`, but auth and admin APIs are largely under `/api` and `/auth`. Access on `/admin` alone would leave login and API exposed. All-path Access already covers `/admin`. Extra admin lock = Tailscale private URL habit + single Immich admin user + registration off.

## Cloudflare ~100MB note

Free/Pro HTTP proxy (including Tunnel) caps request bodies around **100MB**. Large videos on `photos.homz.fyi` may fail to upload. Accepted for this design. Escape hatch (ops only, not required on phones): upload via `photos.maktaba.home` on Tailscale.

## Roles

| Actor | How they connect |
| --- | --- |
| You (admin) | Prefer Tailscale `photos.maktaba.home`; public URL only if needed |
| Household | Immich user accounts; app → `photos.homz.fyi` + CF service-token headers |
| Friends | Temporary Access email allowlist → OTP → Immich share / view |

## Out of scope

- Publishing Vaultwarden on `homz.fyi`
- Cloudflare Access bypass for `/share/*`
- Requiring Tailscale on household phones
- Open Immich registration
- Dedicated NFS move for Immich library (existing later task)

## Success criteria

1. Unallowlisted browser → Access deny (no Immich login page).
2. Allowlisted email + OTP → Immich reachable on public URL.
3. Phone with service tokens → sync/backup works.
4. Admin on Tailscale URL works; vault `/alive` remains OK.
5. Friend email add → access; remove → blocked.
6. `cloudflared` covered by healthcheck; dual URLs documented; friend + token runbooks written.
