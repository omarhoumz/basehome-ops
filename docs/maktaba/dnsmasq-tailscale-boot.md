# dnsmasq + Tailscale boot / auto-restart

dnsmasq serves `*.maktaba.home` and listens on both:

- LAN: `192.168.11.141`
- Tailscale: `100.68.38.53`

Tailscale clients (phone, laptop, remote) use this nameserver via Tailscale split DNS for `maktaba.home`.

---

## Problem

On reboot, systemd starts `dnsmasq` as soon as `network-online.target` is ready. `tailscale0` often does **not** have its IP yet.

dnsmasq then fails with:

```text
failed to create listening socket for 100.68.38.53: Cannot assign requested address
```

After that, `*.maktaba.home` stops resolving until someone restarts dnsmasq by hand.

---

## Installed fix

### 1. Wait script

Path: `/usr/local/bin/wait-for-tailscale-ip.sh`

- Polls `tailscale0` for an IPv4 address for up to **60 seconds**
- Exits `0` when ready, `1` on timeout

```bash
#!/usr/bin/env bash
set -euo pipefail

MAX_WAIT=60

for ((i = 1; i <= MAX_WAIT; i++)); do
  if ip -4 addr show dev tailscale0 2>/dev/null | grep -q 'inet '; then
    exit 0
  fi
  sleep 1
done

echo "Timed out waiting for tailscale0 IPv4 after ${MAX_WAIT}s" >&2
exit 1
```

### 2. systemd drop-in

Path: `/etc/systemd/system/dnsmasq.service.d/tailscale.conf`

```ini
[Unit]
After=tailscaled.service
Wants=tailscaled.service

[Service]
ExecStartPre=/usr/local/bin/wait-for-tailscale-ip.sh
Restart=on-failure
RestartSec=10
```

What this does:

| Setting | Effect |
| --- | --- |
| `After=` / `Wants=` `tailscaled` | Prefer starting after Tailscale |
| `ExecStartPre=` wait script | Do not bind until `tailscale0` has an IP |
| `Restart=on-failure` | If start still fails, retry |
| `RestartSec=10` | Wait 10s between retries |

Config source of truth for listen addresses: `/etc/dnsmasq.d/maktaba.conf` (copy also in `/srv/maktaba/dnsmasq-maktaba.conf`).

---

## Verify after reboot

```bash
systemctl is-active dnsmasq tailscaled
systemctl status dnsmasq --no-pager
dig +short @100.68.38.53 yt.maktaba.home
dig +short @192.168.11.141 player.maktaba.home
curl -s -o /dev/null -w "yt: %{http_code}\n" http://yt.maktaba.home/
```

Expected:

- both services `active`
- dig returns `100.68.38.53`
- yt HTTP `200` (or redirect from player)

---

## Manual recovery (if auto-start still fails)

```bash
# Confirm Tailscale has its IP
ip -4 addr show tailscale0

# Start / restart DNS
sudo systemctl restart dnsmasq
sudo systemctl is-active dnsmasq
```

Useful logs:

```bash
journalctl -u dnsmasq -b --no-pager
journalctl -u tailscaled -b --no-pager | tail -50
```

---

## Re-apply after OS rebuild

```bash
sudo install -m 755 /dev/stdin /usr/local/bin/wait-for-tailscale-ip.sh <<'EOF'
# paste wait script from above
EOF

sudo mkdir -p /etc/systemd/system/dnsmasq.service.d
sudo tee /etc/systemd/system/dnsmasq.service.d/tailscale.conf >/dev/null <<'EOF'
[Unit]
After=tailscaled.service
Wants=tailscaled.service

[Service]
ExecStartPre=/usr/local/bin/wait-for-tailscale-ip.sh
Restart=on-failure
RestartSec=10
EOF

sudo systemctl daemon-reload
sudo systemctl restart dnsmasq
```

---

## Notes

- Exit node is **not** required for DNS. This CT should stay on Tailscale without `--exit-node` unless you deliberately re-enable it for YouTube IP workarounds.
- Do **not** put a `.bak` file under `/etc/dnsmasq.d/` — dnsmasq includes every file in that directory and duplicate keywords will prevent startup.
- If you change the Tailscale IPv4 of `maktaba`, update both `listen-address` and `address=/maktaba.home/...` in the dnsmasq config, then `sudo systemctl restart dnsmasq`.
