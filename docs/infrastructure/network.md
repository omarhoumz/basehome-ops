# Network topology

**Canonical map:** edit [`topology.yaml`](./topology.yaml), then regenerate:

```bash
python3 scripts/render-topology.py
```

| Output | Audience |
| --- | --- |
| [`topology.md`](./topology.md) | Humans + agents (tables, ASCII tree) |
| [`topology.html`](./topology.html) | Humans (scan page + Mermaid) |

Do **not** hand-edit the generated MD/HTML.

## Quick facts

- **LAN:** `192.168.11.0/24` · gateway `192.168.11.1`
- **Proxmox pve:** `192.168.11.123` · TS `100.91.123.54`
- **Kubuntu maktaba VM:** `192.168.11.141` · TS `100.68.38.53`
- **DNS:** Tailscale split DNS · `*.maktaba.home` → `100.68.38.53`
- **Remote access:** Tailscale only — no app ports on the ISP router

## Related

- [usb-wdred-runbook](../maktaba/usb-wdred-runbook.md) — USB + NFS ops
- [vaultwarden remote-access](../vaultwarden/remote-access.md) — VPN options
- [Immich setup plan](../immich/setup-plan.md) — next service (planned)
- [REGISTRY](../../projects/REGISTRY.md) — live service gate
