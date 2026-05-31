# CLAUDE.md

Nextcloud deploy artifacts on 3-node k3s-over-Cloudflare-WARP-Mesh. Not app code. Live `https://nextcloud.itguys.ro` (Mesh-only, :443 via proxy hostPort).

## Operating model: push-to-main CI

- Push main → ARC runner `arc-itguys-ro-nextcloud` (asus-pinned, in-cluster) runs `.github/workflows/deploy.yml`. PR → `pr-checks.yml` dry-runs (no apply, no secrets).
- Runner SA bound to Role `nextcloud-deployer` (ns `nextcloud`, broad CRUD) + Role `cf-token-writer` (ns `cert-manager`, narrow update of Secret `cloudflare-api-token` only). RBAC at `manifests/bootstrap/11-ci-deployer-rbac.yaml`.
- Path filter SKIPS `manifests/60-nginx-tls-proxy.yaml` — shared :443 also serves headlamp/degoog/searxng/degoog-mcp w/ extra vhosts + `/tls-warp` cert mount not in repo. Edit live out-of-band; repo file = historical baseline (header warns DO NOT apply).
- `manifests/bootstrap/` + ARC scale set helm release = one-time manual (CI cannot grant itself rights). Re-apply on fresh cluster.
- `workflow_dispatch` with `reconcile_all=true` forces full reconcile.
- Design: `docs/superpowers/specs/2026-05-25-nextcloud-ci-deployment-design.md`. Original: `docs/2026-05-17-nextcloud-k3s-design.md`.
- Cluster runbook (outside repo): `~/cloudflare-mesh-k3s-state.md` — SoT for node names, Mesh IPs, components.

## Architecture invariants (don't violate without revisiting design)

- **Single-node pin:** whole stack in ns `nextcloud` w/ `nodeSelector: kubernetes.io/hostname=asus-laptop` on every pod. Storage = `local-path` (node-local RWO) on asus only. Nothing schedules to `acer-laptop` or `wsl`. No storage HA — accepted.
- **Mesh-only, no public:** asus nginx TLS proxy binds host `:443` via `hostPort` (single replica; Service ClusterIP). DNS `nextcloud.itguys.ro` → A `100.96.0.2`, DNS-only / grey-cloud. Cluster cloudflared tunnel deliberately NOT used. No ingress controller (traefik disabled). Future apps = added nginx SNI server blocks + own cert.
- **Components:** chart `nextcloud/nextcloud` (Apache img, plain HTTP Service :8080; chart nginx sidecar OFF — TLS terminated by separate front nginx, Deviation #1). Dedicated MariaDB (Postgres rejected). Dedicated Valkey for `memcache.locking`/`memcache.distributed` — fresh instance, do NOT reuse degoog's. TLS via cert-manager (ns `cert-manager`) w/ Cloudflare DNS-01 ClusterIssuer, auto-renewed (proxy self-reloads on rotation).
- **Backups ≠ replication:** nightly asus CronJob `mariadb-dump --single-transaction` (no `occ`/maintenance-mode — Deviation #2) + `config.php` copy to local PVC, then `rsync` data + dump to `acer-laptop` over Mesh SSH. Acer copy = real recovery path; local PVC only covers accidental deletion.

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
