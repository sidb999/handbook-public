# CI: Gitea Actions

Rebuild container images on `git push`, push to the Gitea OCI registry.
Runner is a pod on the **Talos** cluster (gitnode does zero build work —
its M1 SSD is non-removable).

## Components

- Gitea 1.25 (Actions enabled by default — no config needed).
- Runner: Helm chart `giteacharts/actions` **0.1.1** (0.1.0 has a
  storageClass YAML bug), ns `ci` (PSA `privileged`), manifest in
  `infra/k8s` `clusters/talos/ci/`.
  - Pod: `init-gitea` init + `dind` sidecar (docker:29.4.0-dind, privileged,
    socket via emptyDir) + `runner` (gitea/runner:1.0.4, YAML config).
  - Runner label: `talos:docker://golang:1.26` → workflow `runs-on: talos`,
    job container = golang:1.26.
  - 10Gi longhorn-ssd PVC = runner checkout/task cache (NOT docker layers —
    those live in the ephemeral emptyDir and re-pull on pod reschedule).
- Registry: Gitea OCI (`git.example.com`). Push needs a token with scope
  `write:package` (NOT `packages:write`). Stored as Gitea repo secret
  `REGISTRY_PASSWORD` in the app repo.

## Workflow (`.gitea/workflows/build.yaml` in the app repo)

push to master → `runs-on: talos` → golang:1.26 container:
1. `git clone` (plain — see pitfalls)
2. `apt-get install docker.io` (golang image has no CLI; daemon = dind)
3. `docker login git.example.com` (repo secret)
4. `docker build` + push `<sha7>` + `latest`

**No auto-deploy** — the pipeline stops at "image pushed". Deploys are manual
tag bumps in the Flux HelmRelease (option A; see gitops.md).
