# nextcloud (k3s homelab)

CI-deployed Nextcloud stack for the 3-node k3s-over-Cloudflare-WARP-Mesh cluster. Push to `main` triggers `.github/workflows/deploy.yml` on the self-hosted ARC runner `arc-itguys-ro-nextcloud`.

Cluster operational runbook (outside repo): `~/cloudflare-mesh-k3s-state.md`.
Design: `docs/superpowers/specs/2026-05-25-nextcloud-ci-deployment-design.md`.

## Layout

```
.github/workflows/   deploy.yml (push:main) + pr-checks.yml (PR dry-run)
docs/                design specs
helm/                Helm values (committed, secret-free): cert-manager + nextcloud
manifests/           raw k8s YAML (CI-applied: certificate, MariaDB, Valkey, PVCs, backup CronJob)
manifests/bootstrap/ one-time, applied manually by cluster-admin (namespace, ClusterIssuer, deployer RBAC)
secrets/             *.example templates only. Real values live in GitHub Actions Secrets.
```

`manifests/60-nginx-tls-proxy.yaml` is **not CI-applied** — it's a historical baseline of the nextcloud-only vhost. The live proxy is a shared `:443` ingress serving 4 other apps; edit it out-of-band.

## Secrets

Plaintext **never** committed (`.gitignore` enforces).

Workflow on rotation:
1. Update `secrets/<name>.yaml` locally (gitignored).
2. `gh secret set <NAME>` for each affected key (see `.github/workflows/deploy.yml` for the GH secret names).
3. Re-run deploy (`gh workflow run deploy.yml -f reconcile_all=true`).

GitHub Actions Secrets currently set:
`CF_API_TOKEN`, `NEXTCLOUD_ADMIN_USERNAME`, `NEXTCLOUD_ADMIN_PASSWORD`, `MARIADB_ROOT_PASSWORD`, `MARIADB_USERNAME`, `MARIADB_PASSWORD`, `VALKEY_PASSWORD`, `BACKUP_SSH_PRIVATE_KEY`.

The CF API token must be scoped `Zone:DNS:Edit` + `Zone:Zone:Read` on `itguys.ro` only.

## Bootstrap (one-time, per cluster)

Cluster-admin runs these once. CI cannot grant itself rights.

1. `kubectl apply -f manifests/bootstrap/00-namespace.yaml`
2. `helm install cert-manager jetstack/cert-manager -n cert-manager --create-namespace --version v1.20.2 -f helm/cert-manager-values.yaml`
3. Create initial `cloudflare-api-token` Secret in ns `cert-manager` (CI's narrow Role can `update` but not `create`):
   ```
   kubectl -n cert-manager create secret generic cloudflare-api-token --from-literal=api-token=<token>
   ```
4. `kubectl apply -f manifests/bootstrap/10-clusterissuer-letsencrypt.yaml`
5. `kubectl apply -f manifests/bootstrap/11-ci-deployer-rbac.yaml`
6. Install ARC scale set (see design doc §4.2 for the helm install command).
7. `gh secret set` for all 8 secrets (see above).
8. Live-edit the shared nginx-tls-proxy ConfigMap to add the `nextcloud.itguys.ro` vhost (if not already present).
9. DNS `nextcloud.itguys.ro` A → `100.96.0.2` (DNS-only).
10. Cloudflare Gateway "Do Not Inspect" rule for the hostname (see Operational dependencies).
11. Push to `main` → CI takes over.

## Operating

- Routine change: `git push origin main` → CI applies path-relevant jobs. No further action needed; secrets job always runs, manifests/helm jobs run only if their paths changed.
- Rollback: `gh workflow disable deploy.yml`, then `helm rollback nextcloud -n nextcloud` manually.

### Force full reconcile

```
gh workflow run deploy.yml -f reconcile_all=true
```

When to use:
- Drift suspected (someone ran `kubectl edit` / `helm upgrade` against the cluster out-of-band).
- A GH Actions Secret was rotated but no repo file changed (path filter wouldn't trigger helm/manifests).
- Workflow was disabled and re-enabled; first run after re-enable.

**Not needed** after a normal merge — the merge push already triggers all relevant jobs via paths-filter (helm filter includes `.github/workflows/deploy.yml`, so a workflow edit forces helm reconcile too).

Access: `https://nextcloud.itguys.ro` (Mesh participants only).

## Operational dependencies (silent-failure if removed)

- **Cloudflare Gateway "Do Not Inspect" rule** — `itguys` org has Gateway `tls_decrypt` enabled. Rule id `df440536-0b50-483d-b5d7-70cd7cbe6230` (`action: off`, `http.conn.hostname == "nextcloud.itguys.ro"`) exempts this host. If deleted: clients get the Gateway CA, Nextcloud Android app breaks, file traffic decrypted at Cloudflare's edge. Verify: `echo | openssl s_client -connect 100.96.0.2:443 -servername nextcloud.itguys.ro 2>/dev/null | openssl x509 -noout -issuer` must show `O = Let's Encrypt`. Any new host on :443 needs its own rule. Full context: design doc §5 amendment 2026-05-18.
- **PV reclaim policy = Retain** — all three claims (`nextcloud-data`, `nextcloud-db`, `nextcloud-backups`) patched to `persistentVolumeReclaimPolicy: Retain` (local-path defaults to `Delete`). An accidental `kubectl delete pvc` does not wipe the hostPath. Real disk-loss recovery is the nightly acer rsync.
