# Networking

## Topology

- **Mikrotik RB5009** (10.0.0.254) — router, DNS authority for `example.com`.
  Dual WAN: ether1 = optical (primary, DHCP, dist 1), ether2 = STARLINK/LTE
  (Huawei B311-221, backup, static 0/0 dist 2, ping-checked). Masquerade on
  both. Failover/failback verified 2026-08-24.
- **gitnode** — k3s on Mac mini M1, 10.0.0.102. Runs Gitea (`git.example.com`) +
  git hosting. Non-removable SSD — keep its load light.
- **Talos cluster** — 10.0.0.1/.3/.4/.5, all services.

## DNS (Mikrotik)

Mikrotik is the live authority for `example.com`.

- `git.example.com` → 10.0.0.102 (gitnode/Gitea)
- `*.k8s.example.com` → 10.0.0.121 (Envoy Gateway VIP)

The `*.k8s.example.com` wildcard → single gateway VIP works because the gateway
routes by **Host header**, not IP. No per-service DNS rules needed.

## External access

Outside the home network, everything comes in over WireGuard — no ports on
the Mikrotik. Server: VPS-PUBLIC-IP:51820 (Hetzner), tunnel subnet
`10.0.0.0/24`, tunnelhost (10.0.0.3) forwards into the home network.
Full details, client onboarding, and the routing gotchas: [wireguard.md](wireguard.md).