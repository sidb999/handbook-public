# Home infrastructure handbook

Documentation for a self-hosted home infrastructure: a Talos Kubernetes
cluster running production-style services, wired together with GitOps,
monitored end-to-end, and reachable from the internet over WireGuard.

This is a **public mirror** of my internal notes. All hostnames, IPs, and
identifiers have been redacted; the architecture, decisions, and pitfalls are
real.

## What's here

- [Talos cluster](talos-cluster.md) — nodes, disk strategy, Longhorn storage classes
- [GitOps with Flux](gitops.md) — repo layout, SOPS/age secrets, the "add an app" loop
- [Networking](network.md) — router, DNS, gateway, external access
- [WireGuard server](wireguard.md) — the only external access path: onboarding clients, routing rules
- [CI: Gitea Actions](ci.md) — self-hosted CI runner on the cluster (dind), image registry
- [Monitoring](monitoring.md) — Prometheus/Grafana stack, sizing
- [llama host](llama.md) — dedicated LLM inference box (RTX 3090) + agent VM

## Stack at a glance

| Layer | Choice |
|---|---|
| OS / nodes | Talos Linux (immutable), 1 CP + 3 workers |
| GitOps | Flux (GitRepository → Kustomizations → HelmReleases) |
| Secrets | SOPS + age (cluster key in `flux-system`, never on a machine) |
| Storage | Longhorn (per-tier storage classes: ssd/hdd/nvme/backup) |
| Ingress | Envoy Gateway (single VIP, Host-header routing) + MetalLB L2 |
| CI | Gitea Actions, runner pod on the cluster, dind |
| Registry | Gitea OCI (images + Helm charts) |
| Monitoring | kube-prometheus-stack (Prometheus + Grafana + Alertmanager) |
| External access | WireGuard (no open router ports) |

## Design principles

- **Reproducible from Git.** Machine config, disks, and every service are
  declarative; a node can be rebuilt from the repo.
- **One deploy path.** If it's not in the GitOps repo, it's not deployed.
- **Secrets stay encrypted at rest** and the age private key never touches a
  machine or a repo.
- **Minimal configs.** Charts run near-defaults; values are overridden only
  where a real requirement demands it.
