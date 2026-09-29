# WireGuard server

The only way for outside people (friends, remote work) to reach the example.com
intranet (10.0.0.0/24 — Talos, Gitea, grafana, etc.) is through this
WireGuard tunnel. No ports are opened on the Mikrotik; everything goes over
UDP 51820.

## Topology

```
friend (10.0.0.5) ─┐
                    ├── VPS-PUBLIC-IP:51820 ── tunnelhost (10.0.0.3) ── home 10.0.0.0/24
tunnelhost ─────────┘   (Hetzner, Debian 12)
```

- **Tunnel subnet**: `10.0.0.0/24`
- **Server**: VPS-PUBLIC-IP (Hetzner, Debian 12). `wg0` = `10.0.0.1/24`,
  port 51820, NAT via PostUp.
- **tunnelhost** (10.0.0.3, on the home LAN): the forwarder. It's the ONLY
  server peer whose `AllowedIPs` includes the home subnet — that asymmetry
  is what keeps routing sane (see below).

## Server config (`/etc/wireguard/wg0.conf` on the VPS)

```ini
[Interface]
PrivateKey = <on the box, root-only>
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <tunnelhost pub>
AllowedIPs = 10.0.0.3/32, 10.0.0.0/24
PersistentKeepalive = 25

[Peer]
PublicKey = <friend pub>
AllowedIPs = 10.0.0.5/32
PersistentKeepalive = 25
```

Note: this old wireguard-tools (1.0.20210914) build rejects named peer
sections (`[Peer foo]`) — use bare `[Peer]`.

## Adding a new client

1. `wg genkey` → derive pubkey (do this ON the VPS or locally, never
   commit the private key anywhere).
2. Server: add a `[Peer]` block, `AllowedIPs = 10.0.0.N/32` ONLY (no home
   subnet — the loop rule below).
3. `systemctl restart wg-quick@wg0`.
4. Give the client:

```ini
[Interface]
Address = 10.0.0.N/32
MTU = 1420
ListenPort = <any>
PrivateKey = <theirs>

[Peer]
PublicKey = <server-wg-public-key>
AllowedIPs = 10.0.0.0/24, 10.0.0.0/24
Endpoint = VPS-PUBLIC-IP:51820
PersistentKeepalive = 25
```

5. Verify: on the server `sudo wg show` → peer shows `latest handshake`;
   `ping 10.0.0.N`.

## Routing rules (the part that bites)

- **Client `AllowedIPs` must NOT include the server's public IP.**
  If it does, the client routes its own keepalives into wg0 → they loop
  back through the tunnel → the server never learns the client's endpoint
  → no handshake, 0 packets arrive. (Happened 2026-09-29 while debugging;
  symptom: client shows "sent 18 KiB, received 0", server tcpdump sees
  nothing on eth0.)
- **`10.0.0.0/24` appears in exactly one server peer's AllowedIPs**
  (tunnelhost's). If two peers claim the home subnet, the server can't
  pick a route back and traffic black-holes.
- Client-side `10.0.0.0/24` in AllowedIPs = "I want home access".
  Server-side it = "this peer is my gateway to home".

## Operations

- Check: `sudo wg show` on the VPS (handshake age per peer).
- Restart: `sudo systemctl restart wg-quick@wg0`.
- The server keypair lives only in `/etc/wireguard/wg0.conf` (600 root).
- If a new client type appears, reuse the 10.0.0.x space.
