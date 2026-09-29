# GitOps with Flux

Flux v2.9.5 on the Talos cluster. Everything deployed to the cluster is a file
in `infra/k8s` (branch `master`). Flux polls it every minute; no other deploy
path exists.

## Repo layout (`infra/k8s`)

```
clusters/talos/
  kustomization.yaml              # root: lists all app kustomizations
  flux-system/                    # Flux bootstrap (GitRepository, Kustomizations)
  metallb/  metallb-pools/
  cert-manager/
  gateway/  gateway-routes/       # Envoy Gateway + HTTPRoutes/TCPRoutes
  certificates/
  test-app/
  longhorn/
  monitoring/                     # kube-prometheus-stack
  ci/                             # Gitea Actions runner
```

Each app dir: `namespace.yaml`, `helmrepository.yaml`, `helmrelease.yaml`,
`kustomization.yaml`, `<app>-kustomization.yaml` (the Flux Kustomization with
SOPS decryption).

## Adding an app (the loop)

1. New dir `clusters/talos/<app>/` with the files above.
2. Add `<app>/<app>-kustomization.yaml` to the root `kustomization.yaml`.
3. Push to master. Flux reconciles in ~1 min.
4. Verify: `flux get kustomizations`, `flux get helmreleases`, `kubectl -n <app> get pods`.

## SOPS/age

- Secrets are SOPS-encrypted with the **cluster's** age key, which lives in the
  `flux-system/sops-age` secret (Flux pattern) — NOT on any machine.
- Each app dir has a `.sops.yaml` with the age recipient
  (`age1<redacted — cluster age public key>`;
  see `ci/.sops.yaml`).
- The Flux Kustomization has `spec.decryption: {provider: sops, secretRef: sops-age}`.
- Local decrypt (e.g. to inspect): extract the key once
  `kubectl -n flux-system get secret sops-age -o jsonpath='{.data.age\.agekey}' | base64 -d > /tmp/key`
  then `SOPS_AGE_KEY_FILE=/tmp/key sops -d secret.yaml`.
- **Never commit the age private key.** It must be backed up offline.

## Charts

- Charts are stored in the **Gitea OCI registry** (`git.example.com`), not in this repo.
  `helm push <chart>.tgz oci://git.example.com/<you>` (or `oci://git.example.com/infra`).
- HelmRepository for OCI: `type: oci`, url = `oci://<registry>/<namespace>` —
  **do not include the chart name** in the URL; Flux appends it
  (including it → double path → 401/404).
- App images are also in the Gitea OCI registry (CI pushes them, see ci.md).
  The image tag is a plain string in the HelmRelease values — bumping it is
  the deploy.

## Reconcile / debugging

```bash
flux reconcile kustomization <name> --with-source
flux get helmreleases -A
flux logs helmrelease/<name>
kubectl get kustomizations,helmreleases -A
```

If a HelmRelease stalls: delete + recreate it (Flux re-runs from scratch).
