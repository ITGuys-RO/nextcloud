# nextcloud

## Goal

CI-deployed Nextcloud on the homelab k3s cluster — a self-hosted file sync/share for Mesh participants, reached at `https://nextcloud.itguys.ro`. Every push to `main` reconciles the stack; no manual `kubectl`/`helm` against the cluster for routine changes.

## Stack

- **App**: Nextcloud via the official Helm chart `nextcloud/nextcloud`, chart version `9.1.0` (Apache image on plain HTTP :80; chart nginx sidecar OFF — TLS terminated by a separate shared nginx front-proxy).
- **Database**: standalone `mariadb:11.4` (`externalDatabase` mode; bundled bitnami subchart disabled), 20Gi PVC.
- **Cache**: standalone `valkey/valkey:8-alpine` as the Redis backend (`externalRedis`).
- **TLS**: cert-manager `v1.20.2` (jetstack chart) issuing a Let's Encrypt cert via Cloudflare DNS-01 on `itguys.ro`.
- **Storage backend**: k3s `local-path` PVCs — `nextcloud-data` 300Gi, `nextcloud-backups` 100Gi, `nextcloud-db` 20Gi. All three patched to reclaim policy `Retain` (local-path defaults to `Delete`).
- **Node pinning**: whole stack pinned to host `asus-laptop` (`nodeSelector`).

## Repo

`ITGuys-RO/nextcloud` (confirmed: `origin https://github.com/ITGuys-RO/nextcloud.git`).

Layout:
- `.github/workflows/` — `deploy.yml` (push:main) + `pr-checks.yml` (PR dry-run).
- `helm/` — committed, secret-free Helm values (cert-manager + nextcloud).
- `manifests/` — CI-applied raw k8s YAML (certificate, MariaDB, Valkey, PVCs, backup CronJob).
- `manifests/bootstrap/` — one-time, applied manually by cluster-admin (namespace, ClusterIssuer, deployer RBAC).
- `secrets/` — `*.example` templates only; real values live in GitHub Actions Secrets.
- `docs/` — design specs.

## Deploy

CI-deployed via GitHub Actions (`.github/workflows/deploy.yml`). Runs **in-cluster** on the self-hosted ARC runner `arc-itguys-ro-nextcloud`, authenticating to the k3s API through the runner pod's ServiceAccount (Role `nextcloud-deployer` in ns `nextcloud`, Role `cf-token-writer` in ns `cert-manager`).

- Push to `main` → path-filtered jobs: `secrets` always runs (idempotent), `manifests`/`helm` run only when their paths changed.
- Force full reconcile: `gh workflow run deploy.yml -f reconcile_all=true` (drift, rotated secret with no file change, or first run after re-enable).
- Rollback: `gh workflow disable deploy.yml`, then `helm rollback nextcloud -n nextcloud` manually.

Hosted on [Homelab](https://github.com/ITGuys-RO/k3s-cluster/blob/main/docs/homelab.md).

## Status

Active. Latest commit `3a05601` (2026-05-25): readme clarification on when to force reconcile. Stack is bootstrapped and CI-driven; design doc at `docs/superpowers/specs/2026-05-25-nextcloud-ci-deployment-design.md`.

## Tasks

- [ ] Consider a least-privilege MariaDB backup user (current CronJob dumps as root — homelab trade-off noted in `70-backup-cronjob.yaml`).
- [ ] Add `limits.memory` to the Nextcloud Deployment if the asus node shows OOM pressure (currently unbounded by design).
- [ ] Reconcile `manifests/60-nginx-tls-proxy.yaml` history vs. the live shared `:443` proxy (not CI-applied; edited out-of-band).

## Notes

- **Storage**: `local-path` hostPath PVCs on `asus-laptop`, all set to `Retain` so an accidental `kubectl delete pvc` does not wipe data; real disk-loss recovery relies on the nightly rsync to `acer-laptop`.
- **Backups**: nightly CronJob at `03:00` (`70-backup-cronjob.yaml`) — `mariadb-dump` (single-transaction, dump-to-file with size guard to avoid silent empty `.gz`), gzip + `config.php` copy, local 7-day prune, then `rsync` of data + latest dump off-node to `acer-laptop` (`100.96.0.4`). See [Backups & DR](https://github.com/ITGuys-RO/k3s-cluster/blob/main/docs/backups-and-dr.md).
- **Ingress**: served on `:443` by a shared nginx front-proxy serving 4 other apps (not the chart sidecar). DNS `nextcloud.itguys.ro` A → `100.96.0.2` (DNS-only). A Cloudflare Gateway "Do Not Inspect" rule exempts the host from `tls_decrypt`; if removed, clients get the Gateway CA and the Nextcloud Android app breaks.
- **CI runner:** deploy + PR dry-run run on the self-hosted k3s ARC runner `arc-itguys-ro-nextcloud`, built from [shared-workflows](https://github.com/dustfeather/shared-workflows/blob/main/docs/OVERVIEW.md).
- Area: [Homelab](https://github.com/ITGuys-RO/k3s-cluster/blob/main/docs/homelab.md).

## Log

- **2026-05-31** — Note created from repo scan.
