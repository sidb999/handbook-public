# Talos cluster

K8s cluster running all services (Gitea is NOT here — it's on gitnode).

## Facts

- Talos v1.14.0, k8s v1.36.0
- 4 nodes: 1 control plane (10.0.0.1), 3 workers
- CNI: Flannel v0.27.4 (built into Talos image, not independently upgradable)
- PodCIDR: 10.244.0.0/16
- All machine config lives in `infra/bootstrapper` (`patches/` convention) —
  the cluster is reproducible from Git.

## Nodes

| Name | IP | CPU | RAM | Disks |
|---|---|---|---|---|
| CP | 10.0.0.1 | Intel N100, 4c | 32GB | system LITEON LCH-256V2 |
| node-2 | 10.0.0.3 | Xeon E5-2660 v4, 28c | 64GB | 4× MZ7KH240 240GB (SSD), WD140EFGX (HDD), SA400S3 960GB (backup, DRAM-less), SSDPEKNW010T8 (NVMe) |
| node-3 | 10.0.0.4 | Xeon E5-2660 v4, 28c | 64GB | 4× MZ7KH240, WD140EFGX |
| node-4 | 10.0.0.5 | Xeon E5-2660 v4, 28c | 64GB | 4× MZ7KH240 (no HDD) |

**Disk rule: LITEON (LCH-256V2) is the system disk on every node — never touch.**
`sdX` names change across reboots; identify disks by vendor/WWID via
`talosctl get disks`.

Hardware notes:
- CP is a low-power mini (N100); workers are E5 v4 machines — CPU-heavy
  workloads land on workers, CP just runs control-plane + etcd.
- E5-2660 v4: 14 cores × 2 threads (Haswell, 2016) — fast enough for
  everything we run; DDR4 ECC.
- SA400S3 960GB: DRAM-less, sustained-write cliff ~100–150 MB/s — backup
  tier only, never for data that needs random I/O.
- MZ7KH240: SATA SSD, the workhorse tier (default Longhorn pool).
- SSDPEKNW010T8: 1TB NVMe, only on node-2 — the only NVMe disk in the
  cluster, so `longhorn-nvme` (2 replicas) can't keep both replicas on NVMe.

## Longhorn (1.12.1)

16 default disks, `storageReservedPercentageForDefaultDisk: 10`.

Tags: `ssd`, `hdd`, `nvme`, `backup`.

| StorageClass | Selector | Replicas |
|---|---|---|
| `longhorn` | (default) | 2 |
| `longhorn-static` | static | 2 |
| `longhorn-ssd` | ssd | 2 |
| `longhorn-hdd` | hdd | 2 |
| `longhorn-nvme` | nvme | 2 |
| `longhorn-backup` | backup | 1 |

`defaultDataLocality: best-effort` (prefer local, allow cross-node).

Notes:
- Longhorn wants **mounted filesystems**, not raw block devices.
- Disk `storageReserved`/tags are snapshotted at creation time — changing
  settings requires disk recreation (disable → diskUpdate {} → controller
  recreates). See `infra/bootstrapper` for the procedure.
- SA400S3 is DRAM-less: sustained-write cliff ~100–150 MB/s. Backup tier only.
- Frontend LB IP: 10.0.0.122.

## LoadBalancer IPs (MetalLB L2, pool 10.0.0.121–200)

| Service | IP |
|---|---|
| Envoy Gateway | 10.0.0.121 (pinned) |
| longhorn-frontend | 10.0.0.122 |
| test-app | 10.0.0.123 |

## Access

```bash
talosctl get machines            # ~/.talos/config
kubectl config use-context admin@talos-cluster-1
```

Talos secure ports (6443 etc.) are NOT exposed — some kubelet/metrics targets
scraping CP secure ports are expected-down.
