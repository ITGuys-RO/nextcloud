# manifests/

Raw Kubernetes YAML.

## CI-applied (push to main → `.github/workflows/deploy.yml`)

Applied by the `nextcloud-deployer` ServiceAccount in ns `nextcloud`:

- `40-valkey.yaml`
- `50-nextcloud-pvcs.yaml`
- `70-backup-cronjob.yaml`

## TLS is not terminated in this namespace

`edge-proxy` (ns `ingress`, owned by the `k3s-cluster` repo) binds host `:443`
on acer via `hostPort` and terminates TLS for every mesh vhost with its own
`edge-mesh-tls` certificate, whose SAN list carries `cloud.itguys.ro`. Adding
or renaming a vhost means editing edge-proxy there, not adding anything here.

The per-namespace `nginx-tls-proxy` this repo used to carry was retired in the
2026-08-08 asus decommission (two of them would have collided on hostPort 443
on the surviving node) and its manifest was deleted on 2026-09-03 along with
the `nextcloud-tls` Certificate and the `nextcloud-https` Service it defined.
It is in git history if it is ever needed.

## bootstrap/ (one-time, applied manually by cluster-admin)

- `bootstrap/00-namespace.yaml` — creates ns `nextcloud`.
- `bootstrap/10-clusterissuer-letsencrypt.yaml` — cluster-scoped, references
  `cloudflare-api-token` in ns `cert-manager`.
- `bootstrap/11-ci-deployer-rbac.yaml` — `nextcloud-deployer` SA + Role +
  RoleBinding in ns `nextcloud`, plus `cf-token-rotator` SA + ClusterRole +
  RoleBinding in ns `cert-manager`, plus the ARC runner SA impersonation
  binding.
