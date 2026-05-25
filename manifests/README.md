# manifests/

Raw Kubernetes YAML.

## CI-applied (push to main → `.github/workflows/deploy.yml`)

Applied by the `nextcloud-deployer` ServiceAccount in ns `nextcloud`:

- `20-certificate.yaml`
- `30-mariadb.yaml`
- `40-valkey.yaml`
- `50-nextcloud-pvcs.yaml`
- `70-backup-cronjob.yaml`

## Informational only — NOT applied by CI

- `60-nginx-tls-proxy.yaml` — shared `:443` ingress on `asus-laptop` also serves
  `headlamp.itguys.ro`, `degoog.itguys.ro`, `searxng.itguys.ro`,
  `degoog-mcp.itguys.ro`. Live state has additional vhosts and a second cert
  mount (`/tls-warp`, secret `mesh-warp-tls`) not declared here. Managed
  out-of-band. The repo copy is a historical baseline of the nextcloud-only
  vhost; do not `kubectl apply` it as-is.

## bootstrap/ (one-time, applied manually by cluster-admin)

- `bootstrap/00-namespace.yaml` — creates ns `nextcloud`.
- `bootstrap/10-clusterissuer-letsencrypt.yaml` — cluster-scoped, references
  `cloudflare-api-token` in ns `cert-manager`.
- `bootstrap/11-ci-deployer-rbac.yaml` — `nextcloud-deployer` SA + Role +
  RoleBinding in ns `nextcloud`, plus `cf-token-rotator` SA + ClusterRole +
  RoleBinding in ns `cert-manager`, plus the ARC runner SA impersonation
  binding.
