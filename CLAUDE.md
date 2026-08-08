# CLAUDE.md

Nextcloud deploy artifacts on k3s-over-Cloudflare-WARP-Mesh. Not app code. Live `https://nextcloud.itguys.ro` (Mesh-only, :443 via `edge-proxy` in ns `ingress`).

> **2026-08-08 asus-laptop decommission.** Cluster is now 2 nodes: `acer-laptop` (control-plane + only app node) and `wsl`. Everything below that used to say asus now says acer. See commit `b793012`.

## Operating model: push-to-main CI

- Push main → ARC runner `arc-itguys-ro-nextcloud` (in-cluster, ns `arc-runners`, on acer) runs `.github/workflows/deploy.yml`. PR → `pr-checks.yml` dry-runs (no apply, no secrets).
- **`kubectl apply --dry-run=client` is NOT offline.** Apply GETs the live object to build the merge patch, so every dry-run needs a namespace the deployer SA can read. `manifests/*.yaml` carry `metadata.namespace: nextcloud`; `helm template` output does NOT, so its dry-run must pass `-n "${KUBE_NS}"` explicitly. Omit it and kubectl defaults to the runner pod's own ns `arc-runners` → `Forbidden` (broke run 31261070925).
- Runner SA bound to Role `nextcloud-deployer` (ns `nextcloud`, broad CRUD) + Role `cf-token-writer` (ns `cert-manager`, narrow update of Secret `cloudflare-api-token` only). RBAC at `manifests/bootstrap/11-ci-deployer-rbac.yaml`.
- Path filter SKIPS `manifests/60-nginx-tls-proxy.yaml` — **retired 2026-08-08**. That proxy is gone; shared Mesh :443 is now `edge-proxy` (ns `ingress`), owned by the k3s-cluster repo. File kept as historical record only; applying it would fight edge-proxy for hostPort 443.
- `manifests/bootstrap/` + ARC scale set helm release = one-time manual (CI cannot grant itself rights). Re-apply on fresh cluster.
- `workflow_dispatch` with `reconcile_all=true` forces full reconcile.
- Design: `docs/superpowers/specs/2026-05-25-nextcloud-ci-deployment-design.md`. Original: `docs/2026-05-17-nextcloud-k3s-design.md`.
- **Cluster itself lives in a separate repo: `ITGuys-RO/k3s-cluster`.** Owns nodes, bootstrap, ARC, cert-manager, and the shared `edge-proxy` ingress (`manifests/edge-proxy/`). Start any cluster-level diagnosis there, not here — this repo only owns ns `nextcloud`.
- Cluster runbook (outside repo): `~/cloudflare-mesh-k3s-state.md` — SoT for node names, Mesh IPs, components.

## Architecture invariants (don't violate without revisiting design)

- **Single-node pin:** whole stack in ns `nextcloud` w/ `nodeSelector: kubernetes.io/hostname=acer-laptop` on every pod. Storage = `local-path` (node-local RWO) on acer only. Nothing schedules to `wsl`. No storage HA — accepted.
- **Mesh-only, no public:** TLS terminated by `edge-proxy` (ns `ingress`, k3s-cluster repo) binding host `:443` via `hostPort` on acer. DNS `nextcloud.itguys.ro` → A `100.96.0.4` (acer Mesh IP), DNS-only / grey-cloud. Cluster cloudflared tunnel deliberately NOT used. No ingress controller (traefik disabled). Adding a vhost = edit edge-proxy in the k3s-cluster repo, not here.
- **Components:** chart `nextcloud/nextcloud` (Apache img, plain HTTP Service :8080; chart nginx sidecar OFF — TLS terminated by separate front nginx, Deviation #1). Dedicated MariaDB (Postgres rejected). Dedicated Valkey for `memcache.locking`/`memcache.distributed` — fresh instance, do NOT reuse degoog's. TLS via cert-manager (ns `cert-manager`) w/ Cloudflare DNS-01 ClusterIssuer, auto-renewed (proxy self-reloads on rotation).
- **Backups SUSPENDED 2026-08-08.** `manifests/70-backup-cronjob.yaml` has `suspend: true`. It existed for the off-node copy: nextcloud ran on asus and rsynced data + latest dump to acer. Now that nextcloud runs *on* acer, the rsync target `100.96.0.4` is the same machine and the same disk — it protects against nothing. **There is currently no backup at all.** The nightly `mariadb-dump --single-transaction` (no `occ`/maintenance-mode — Deviation #2) + `config.php` copy was *not* redundant (point-in-time protection ≠ disk redundancy); unsuspend with the rsync steps dropped if dumps are wanted back without a real off-node target.

## Secrets policy (hard rule — repo may go to GitHub)

- Never commit plaintext. `.gitignore` enforces — do NOT loosen.
- Source-of-truth = GitHub Actions repo Secrets. Workflow renders k8s Secret manifests at apply time. Local `secrets/*.yaml` (gitignored) only used to seed `gh secret set`.
- Commit `secrets/<name>.example` (placeholders) as schema reference.
- CF API token scope: `Zone:DNS:Edit` + `Zone:Zone:Read` on `itguys.ro` only.
- Non-k8s dep: out-of-band Cloudflare Gateway "Do Not Inspect" rule for `nextcloud.itguys.ro` (see README "Operational dependencies" / design §5). Silent failure if removed: clients get Gateway CA cert, Android app breaks, traffic decrypted at CF edge.

<!-- code-review-graph MCP tools -->
## MCP Tools: code-review-graph

**IMPORTANT: This project has a knowledge graph. ALWAYS use the
code-review-graph MCP tools BEFORE using Grep/Glob/Read to explore
the codebase.** The graph is faster, cheaper (fewer tokens), and gives
you structural context (callers, dependents, test coverage) that file
scanning cannot.

### When to use graph tools FIRST

- **Exploring code**: `semantic_search_nodes` or `query_graph` instead of Grep
- **Understanding impact**: `get_impact_radius` instead of manually tracing imports
- **Code review**: `detect_changes` + `get_review_context` instead of reading entire files
- **Finding relationships**: `query_graph` with callers_of/callees_of/imports_of/tests_for
- **Architecture questions**: `get_architecture_overview` + `list_communities`

Fall back to Grep/Glob/Read **only** when the graph doesn't cover what you need.

### Key Tools

| Tool | Use when |
| ------ | ---------- |
| `detect_changes` | Reviewing code changes — gives risk-scored analysis |
| `get_review_context` | Need source snippets for review — token-efficient |
| `get_impact_radius` | Understanding blast radius of a change |
| `get_affected_flows` | Finding which execution paths are impacted |
| `query_graph` | Tracing callers, callees, imports, tests, dependencies |
| `semantic_search_nodes` | Finding functions/classes by name or keyword |
| `get_architecture_overview` | Understanding high-level codebase structure |
| `refactor_tool` | Planning renames, finding dead code |

### Workflow

1. The graph auto-updates on file changes (via hooks).
2. Use `detect_changes` for code review.
3. Use `get_affected_flows` to understand impact.
4. Use `query_graph` pattern="tests_for" to check coverage.
