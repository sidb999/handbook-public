# Monitoring

kube-prometheus-stack (chart 91.4.1) in ns `monitoring`, Flux-managed
(`infra/k8s` `clusters/talos/monitoring/`, HelmRelease `kps`).

## Stack

- Prometheus 3.14 — 20Gi longhorn-ssd, **90d retention**, 10GiB TSDB cap
  (~200MB/day at ~35k series).
- Grafana 13.2.2 — `https://grafana.k8s.example.com`.
- Alertmanager 0.34 — chart default config.
- node-exporter v1.12.1 ×4, cAdvisor, kube-state-metrics, kubelet.
- 14 ServiceMonitors, 32 targets (26 up; 6 down = Talos CP secure ports,
  expected).

## Notes

- Prometheus scrape config is gzipped YAML in secret
  `prometheus-kps-kube-prometheus-stack-prometheus`, key `prometheus.yaml.gz`.
- Histograms: query with `{__name__=~".*_bucket$"}` (not `{name=~...}`).
- Talos secure ports (6443 etc.) are not exposed — CP kubelet targets are
  expected-down.
