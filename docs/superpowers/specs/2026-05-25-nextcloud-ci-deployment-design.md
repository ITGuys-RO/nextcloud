# Design: Nextcloud CI/CD via GitHub Actions self-hosted runner

**Date:** 2026-05-25
**Status:** Approved — ready for implementation
**Predecessor:** `docs/2026-05-17-nextcloud-k3s-design.md` (initial manual deploy)
**Tracking:** NEXT-1

## 1. Goal

Replace the manual `helm upgrade -f ...` / `kubectl apply -f ...` deploy loop with a push-to-main GitHub Actions workflow running on a self-hosted ARC runner inside the k3s cluster. Repo (`ITGuys-RO/nextcloud`) becomes the deployment trigger; merging to `main` reconciles cluster state.

All cluster invariants from the original design survive unchanged:
- Single-namespace pin to `nodeSelector: kubernetes.io/hostname=asus-laptop`
- Mesh-only, no public ingress; nginx TLS proxy on asus `:443` hostPort
- Dedicated MariaDB + Valkey; chart-bundled subcharts off
- Nightly backup CronJob → rsync to acer
- Cloudflare Gateway Do-Not-Inspect rule (out-of-band)

## 2. Non-goals

- GitOps controller (Argo / Flux). Push-to-main is the trigger; the runner is the agent. No drift reconciliation outside of a triggered run.
- Multi-environment promotion (staging → prod). Single homelab; main → cluster, end of story.
- Image building. Nextcloud uses upstream images only.
- Helm chart vendoring or pinning beyond the existing values file. The chart version stays declared in the workflow (one place to bump).
- Secret rotation automation. Rotation = update GH Secret, re-run workflow. Manual cadence.

## 3. Decisions (locked in brainstorming)

| # | Decision | Choice | Why |
|---|---|---|---|
| D1 | Secrets in CI | Stored as GitHub Actions repo Secrets; workflow renders Secret manifests and `kubectl apply`s them | Plaintext never in git; no new in-cluster controller; accepts GitHub IAM coupling as homelab-grade trade-off |
| D2 | Trigger | `push: branches: [main]` → full apply | Simplest deployable-main model; PR review is the gate |
| D3 | Apply scope | Path-filtered jobs (`helm/**` vs `manifests/**` vs `secrets-workflow`) | Faster typical run; drift correction handled by `workflow_dispatch` of full reconcile |
| D4 | Cluster auth | In-cluster ServiceAccount with namespaced RBAC, default in-pod kubeconfig | Token never leaves cluster; minimal blast radius |
| D5 | Bootstrap | ARC scale set + SA/Role/RoleBinding + namespace + ClusterIssuer all applied manually once | One-time, requires cluster-admin; documented, not CI'd |
| D6 | Backup CronJob | Same path filter as other manifests | No asymmetry |
| D7 | First-run safety | Pre-flight: reconcile repo vs live cluster before flipping the switch | First CI run should be a near-no-op; surfaces drift the operator must merge into repo first |

## 4. Architecture

### 4.1 Repository layout (after restructure)

```
.github/
  workflows/
    deploy.yml         # push:main → path-filtered apply
    pr-checks.yml      # PR → helm template + kubectl diff (no apply)

manifests/
  bootstrap/           # NEW: applied once, by hand, by cluster-admin
    00-namespace.yaml          (moved from manifests/)
    10-clusterissuer-letsencrypt.yaml (moved from manifests/)
    11-ci-deployer-rbac.yaml   (NEW: SA + Role + RoleBinding)
  20-certificate.yaml          (unchanged paths)
  30-mariadb.yaml
  40-valkey.yaml
  50-nextcloud-pvcs.yaml
  60-nginx-tls-proxy.yaml
  70-backup-cronjob.yaml

helm/                  # unchanged
secrets/               # *.example only; *.yaml stays gitignored, no longer applied
docs/                  # this spec
```

### 4.2 ARC runner (one-time bootstrap)

Repo-scoped scale set `arc-itguys-ro-nextcloud` in ns `arc-runners`, matching the established pattern (`arc-itguys-ro-degoog-infra`, `arc-itguys-ro-fleet-manager`, …):

- Chart: `oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set` v0.14.1
- `containerMode: { type: dind }`
- `githubConfigSecret: github-app-itguys` (already exists in cluster)
- `githubConfigUrl: https://github.com/ITGuys-RO/nextcloud`
- `minRunners: 0`, `maxRunners: 2`
- Listener + runner pods pinned to `asus-laptop` (wsl outbound flaky; taint imperative-only)

Repo-scoped (not the org-scoped `arc-itguys-ro` set) because GitHub-side runner-group routing for org-owned repos has not been resolved (jobs sit unrouted). Matches the workaround documented in degoog's K3S_MIGRATION_PLAN.md.

### 4.3 ServiceAccount + RBAC (`manifests/bootstrap/11-ci-deployer-rbac.yaml`)

- `ServiceAccount/nextcloud-deployer` in ns `nextcloud`
- `Role/nextcloud-deployer` in ns `nextcloud` — verbs `get,list,watch,create,update,patch,delete` on the chart's resource kinds: `deployments,statefulsets,replicasets,pods,services,configmaps,secrets,persistentvolumeclaims,jobs,cronjobs,serviceaccounts,roles,rolebindings,ingresses,certificates.cert-manager.io`. Plus `events` get/list/watch for `kubectl rollout status` and helm hooks.
- `RoleBinding/nextcloud-deployer` binding the SA to the Role.
- **No ClusterRole.** The SA cannot create/modify the Namespace, ClusterIssuer, or anything outside ns `nextcloud`. Those stay in `bootstrap/`.

The runner pod inside the cluster uses its own SA by default. To make the deploy job run as `nextcloud-deployer`, the workflow uses a one-time `kubectl --token=$(cat /var/run/secrets/...)` is not necessary — instead, the ARC runner pod spec mounts a *projected* token for `nextcloud-deployer` via the workflow step using `kubectl --as=system:serviceaccount:nextcloud:nextcloud-deployer` impersonation. ARC's default runner SA must permit impersonation of `nextcloud-deployer` only, via a narrow ClusterRole `nextcloud-deployer-impersonator` bound to the ARC runner SA in `arc-runners`. This stays in `manifests/bootstrap/11-ci-deployer-rbac.yaml`.

(Alternative considered: mount a kubeconfig secret. Rejected — adds a secret to rotate. Impersonation keeps the credential in cluster.)

### 4.4 Secrets (D1)

GitHub Actions repo Secrets (set out-of-band via `gh secret set`):

| GH Secret | Renders into k8s Secret | Key(s) |
|---|---|---|
| `CF_API_TOKEN` | `cloudflare-api-token` (ns `cert-manager`) | `api-token` |
| `NEXTCLOUD_ADMIN_USERNAME` (default `admin`) + `NEXTCLOUD_ADMIN_PASSWORD` | `nextcloud-admin` (ns `nextcloud`) | `nextcloud-username`, `nextcloud-password` |
| `MARIADB_ROOT_PASSWORD` + `MARIADB_USERNAME` (default `nextcloud`) + `MARIADB_PASSWORD` | `nextcloud-db` (ns `nextcloud`) | `mariadb-root-password`, `db-username`, `db-password` |
| `VALKEY_PASSWORD` | `valkey-auth` (ns `nextcloud`) | `redis-password` |
| `BACKUP_SSH_PRIVATE_KEY` | `backup-ssh` (ns `nextcloud`) | `id_ed25519` |

**Special case — `cloudflare-api-token` is in ns `cert-manager`.** The `nextcloud-deployer` SA has no access there. The workflow handles this by applying the CF secret using a separate job running as a second SA `cf-token-rotator` in ns `cert-manager`, restricted to `get,patch,update` on Secrets named `cloudflare-api-token` only (via `resourceNames`). That ClusterRole + RoleBinding ships in `manifests/bootstrap/11-ci-deployer-rbac.yaml`.

Secrets workflow strategy: secrets are applied on every workflow run (cheap, idempotent — `kubectl apply` with unchanged content is a no-op). No drift detection.

### 4.5 Workflow: `deploy.yml`

```
on:
  push:
    branches: [main]
    paths:
      - 'helm/**'
      - 'manifests/**'
      - '.github/workflows/deploy.yml'
  workflow_dispatch:
    inputs:
      reconcile_all:
        description: "Apply everything (helm + manifests + secrets) regardless of paths"
        type: boolean
        default: false

concurrency:
  group: deploy
  cancel-in-progress: false       # never cancel an in-flight apply
```

**Jobs:**

1. **`detect-changes`** — `dorny/paths-filter` to set outputs: `helm`, `manifests`, `secrets` (always true; runs every workflow).
2. **`apply-secrets`** (always runs) — renders all 5 Secret manifests from GH Secrets via `envsubst` on heredoc templates, `kubectl apply -f -`. Uses `--as=system:serviceaccount:cert-manager:cf-token-rotator` for the CF token; default SA (nextcloud-deployer impersonation) for the rest.
3. **`apply-manifests`** — `if: needs.detect-changes.outputs.manifests == 'true' || inputs.reconcile_all`. Runs `kubectl apply -f manifests/ --recursive=false` (top-level only; bootstrap/ excluded). Streams `kubectl rollout status` per Deployment that changed.
4. **`helm-upgrade`** — `if: needs.detect-changes.outputs.helm == 'true' || inputs.reconcile_all`. Runs `helm upgrade --install nextcloud nextcloud/nextcloud --version <pinned> -f helm/nextcloud-values.yaml -n nextcloud --atomic --timeout 5m`.
5. **`smoke-test`** (depends on apply-* + helm-upgrade) — `curl -k https://nextcloud.itguys.ro/status.php` from inside the runner pod via the Mesh; asserts HTTP 200 and `installed=true` in JSON. Fails the build on regression.

All jobs `runs-on: arc-itguys-ro-nextcloud`. No `actions/checkout` token rotation needed (default `GITHUB_TOKEN` read-only is fine — no pushes back to the repo). Third-party actions pinned by SHA (matches degoog convention).

### 4.6 Workflow: `pr-checks.yml`

```
on:
  pull_request:
    paths: ['helm/**','manifests/**','.github/workflows/deploy.yml','.github/workflows/pr-checks.yml']
```

Single job, same runner:
- `helm template nextcloud nextcloud/nextcloud --version <v> -f helm/nextcloud-values.yaml | kubectl apply --dry-run=server -f -`
- `kubectl apply --dry-run=server -f manifests/`
- `kubectl diff -f manifests/ || true` (advisory, doesn't fail PR — diff output posted in step log)

No secret access from PR runs. Read-only RBAC SA: same `nextcloud-deployer` token suffices because `apply --dry-run=server` only needs the verbs the SA already has.

**Fork PRs:** repo is private under `ITGuys-RO`. Fork PRs treated as not-trusted by default GitHub rules; `pull_request_target` is NOT used (no need; no public collaborators). Standard `pull_request` event only.

### 4.7 Data flow

```
Operator → git push main
              │
              ▼
GitHub Actions (deploy.yml)
              │
              ▼
arc-itguys-ro-nextcloud runner pod (in cluster, asus-pinned)
              │
              │  default SA: ServiceAccount/nextcloud-arc-runner (arc-runners ns)
              │  impersonates: nextcloud-deployer (nextcloud) OR cf-token-rotator (cert-manager)
              ▼
k3s API server
              │
              ▼
ns nextcloud (helm-managed + raw manifests) + ns cert-manager (CF token only)
```

No outbound. The runner reaches the API server via `kubernetes.default.svc.cluster.local:443` (in-cluster).

## 5. Error handling

- **Helm upgrade fails:** `--atomic --timeout 5m` rolls back to last good release. Workflow step exits non-zero; smoke-test still runs and likely fails too (clear signal).
- **kubectl apply fails mid-batch:** apply is per-file; `kubectl rollout status` after each Deployment surfaces stuck pods within 60s. Job fails. Operator inspects via headlamp / `kubectl describe`.
- **Smoke test 5xx:** workflow red. Last-good release stays running (atomic). Operator triages.
- **ARC runner pod doesn't show up:** ARC listener pod in `arc-runners` logs the listener registration error. Workflow shows `queued` for >10 min → operator checks `kubectl -n arc-runners get pods` and the listener logs.
- **Secret rendering produces empty value:** `envsubst` template uses `${VAR?missing}` so an unset GH secret aborts the step rather than writing an empty Secret.

## 6. Testing

Unit-test scope = nothing (no app code). Integration test = the smoke-test job.

Pre-merge: PR check confirms `helm template` + `kubectl --dry-run=server` succeed.
Post-merge: smoke test asserts the live endpoint.

Manual verification gates:
- After bootstrap: `kubectl auth can-i ... --as=system:serviceaccount:...` for each verb the workflow uses.
- After first CI run: confirm `helm history nextcloud -n nextcloud` shows release marked Deployed without rolling back.

## 7. Migration sequence

Listed in order; each is a separate task in the implementation plan.

1. **Pre-flight reconciliation** (T2 in plan). `helm get values nextcloud -n nextcloud` vs `helm/nextcloud-values.yaml`; `kubectl diff -f manifests/` against live. Resolve any drift into the repo. First CI run must be a near-no-op.
2. **Restructure manifests/** (T3). git-mv 00 and 10 into `bootstrap/`. Update `README.md` apply order.
3. **Author RBAC manifest** (T4). Commit only — no apply yet.
4. **Author workflows** (T5, T6). Commit only.
5. **Bootstrap ARC + RBAC** (T7). `helm install arc-itguys-ro-nextcloud ...`; `kubectl apply -f manifests/bootstrap/11-ci-deployer-rbac.yaml`. Verify runner registers with GitHub.
6. **Populate GH Secrets** (T8). `gh secret set` for each, sourced from the local `secrets/*.yaml` files (still gitignored).
7. **Update docs** (T9). `README.md` apply-order section flipped to "Updates are pushed to main; CI reconciles."  `CLAUDE.md` "no GitOps controller; manual `helm upgrade`" line replaced with the new model.
8. **Cutover** (T10). Open PR `feat/github-actions-deploy` → rebase-merge (rebase-only org rule). First CI run = no-op confirmation. If green, the goal is achieved.

## 8. Rollback

- Disable workflows: `gh workflow disable deploy.yml`. The cluster keeps running whatever was last applied — nothing is removed. Operator reverts to manual `helm upgrade`.
- The bootstrap manifests (SA, RBAC, ARC scale set) can be left in place; they cost nothing while idle (minRunners=0).

## 9. Open questions

None at design time. Decisions D1–D7 settle the substantive forks; everything else is mechanical execution.
