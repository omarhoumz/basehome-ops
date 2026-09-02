# Remote Access

Vaultwarden runs on the LAN at `https://vault.maktaba.home`.
To reach it from outside your home network, use a VPN — do not expose Vaultwarden directly to the internet.

---

## Which option to use

| Option | When to use |
|---|---|
| **Tailscale** (default) | New device can install the Tailscale app — already running on this VM |
| **WireGuard** (plain) | Device cannot install Tailscale; you can port-forward UDP 51820 on the router |
| **Headscale** | You want Tailscale client UX but self-hosted control plane (future) |
| **NetBird** | You want a self-hosted mesh platform, not tied to Tailscale (future) |

**Recommendation:** use Tailscale now. Set up plain WireGuard only as a fallback. Defer Headscale/NetBird unless you want to migrate off Tailscale SaaS.

---

## Comparison

| | Tailscale | WireGuard | Headscale | NetBird |
|---|---|---|---|---|
| Tunnel | WireGuard | WireGuard | WireGuard | WireGuard |
| Control plane | Tailscale cloud | None (manual) | You host | You host |
| Client apps | Tailscale | WireGuard app | Tailscale app* | NetBird app |
| Port-forward needed | Usually no | Yes (UDP 51820) | Usually no | Usually no |
| Per-device setup | Install + sign in | Manual config file | Install + join your server | Install + join your network |
| Ops burden | Lowest | Low but manual | Medium–high | Medium–high |
| Third-party dependency | Tailscale Inc. | None | Your server only | Your server only |

\*Headscale is a self-hosted server compatible with Tailscale clients.

---

## Network topology

```
Internet → ISP router → RJ45 → Proxmox host → KUbuntu VM (Docker)
                                                    └─ Caddy → Vaultwarden
```

---

## Option A — Tailscale (recommended, already active)

Tailscale is already running on this VM (`maktaba`, Tailscale IP `100.68.38.53`).
`*.maktaba.home` DNS is handled by maktaba dnsmasq and resolves to the Tailscale IP.

### New device setup

1. Install the Tailscale app (Android, iOS, Windows, macOS, Linux).
2. Sign in to the same Tailscale account.
3. Open `https://vault.maktaba.home` in browser or set it as the Bitwarden self-hosted server URL.

If DNS does not resolve on the remote device, use the Tailscale IP directly:

```
https://<tailscale-ip-of-vm>
```

Or add to the client's hosts file:

```
100.68.38.53  vault.maktaba.home
```

No changes to Vaultwarden or Caddy are needed.

---

## Option B — Self-hosted WireGuard

For devices that cannot install Tailscale. Requires router port-forwarding.

### Requirements

- ISP router: forward **UDP 51820** → KUbuntu VM LAN IP.
- Static public IP or DDNS (e.g. [DuckDNS](https://www.duckdns.org/)).

### Server setup (KUbuntu VM)

```bash
sudo apt install wireguard

wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key
sudo chmod 600 /etc/wireguard/server_private.key
```

Create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <server_private_key>
PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey  = <client_public_key>
AllowedIPs = 10.8.0.2/32
```

Enable:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo systemctl enable --now wg-quick@wg0
```

### Client config (one per device)

```ini
[Interface]
Address    = 10.8.0.2/32
PrivateKey = <client_private_key>

[Peer]
PublicKey  = <server_public_key>
Endpoint   = <your-public-ip-or-ddns>:51820
AllowedIPs = 10.8.0.0/24
PersistentKeepalive = 25
```

Once connected, access Vaultwarden at `https://vault.maktaba.home` (if DNS works over VPN) or `https://10.8.0.1`.

See also [operations.md](operations.md) for trusting the Caddy local CA on the client device.

---

## Option C — Headscale (future, optional)

Self-hosted Tailscale-compatible control server. Use if you want to migrate off Tailscale SaaS while keeping Tailscale client apps.

High-level steps:

1. Deploy Headscale on the KUbuntu VM (Docker recommended).
2. Optionally deploy a self-hosted DERP relay for NAT traversal.
3. Configure namespaces, pre-auth keys, MagicDNS, ACLs.
4. Point Tailscale clients at your Headscale server URL instead of `login.tailscale.com`.
5. Verify `https://vault.maktaba.home` over the mesh.

Trade-offs: you own all identity/policy data, but you operate Headscale + DERP and some Tailscale client features may lag.

Not implemented yet — revisit if you decide to leave Tailscale SaaS.

---

## Option D — NetBird (future, optional)

Open-source mesh VPN with its own client apps (not Tailscale-compatible).

High-level steps:

1. Deploy NetBird management server on the KUbuntu VM or Proxmox.
2. Create network and access policies.
3. Install NetBird client on each remote device.
4. Verify connectivity to Vaultwarden.

Trade-offs: independent ecosystem with OAuth/SSO options, but separate clients and another platform to operate.

Not implemented yet — revisit if you want mesh VPN without the Tailscale ecosystem.

---

## Verification (any path)

From a remote device on the VPN:

1. Open `https://vault.maktaba.home` — page loads (accept/trust cert if needed).
2. Log in to Vaultwarden / Bitwarden client with your account.
3. On the server, run `./healthcheck.sh` — all checks pass.
