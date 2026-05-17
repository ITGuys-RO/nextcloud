# nextcloud (k3s homelab)

Lightweight **versioned-imperative** repo for the Nextcloud deployment on the
3-node k3s-over-Cloudflare-WARP-Mesh cluster. No GitOps controller — changes
are applied manually (`helm upgrade -f ...`, `kubectl apply -f ...`) but every
declarative artifact is version-controlled here.

Cluster operational runbook / source-of-truth: `~/cloudflare-mesh-k3s-state.md`.

## Layout

```
docs/        design spec(s)
helm/        Helm values (committed, secret-free — placeholders only)
manifests/   raw k8s YAML (namespace, NodePort svc, cert-manager issuer,
             backup CronJob, etc.)
secrets/     templates ONLY (*.example). Real secrets are gitignored and
             applied out-of-band.
```

## Secrets policy (read before committing anything)

**Never commit plaintext secrets.** This repo may end up on GitHub.
Excluded by `.gitignore`: everything in `secrets/` except `*.example`, plus
`*-secret.yaml`, `*.key`, `*.pem`, `kubeconfig*`, `.env*`, `*-token*`.

Workflow:
1. A template lives at `secrets/<thing>.example` (placeholders, committed).
2. Copy → `secrets/<thing>.yaml` (real values, **gitignored**).
3. `kubectl apply -f secrets/<thing>.yaml` out-of-band.

Secrets this deployment needs (all out-of-band, never in git):
- Cloudflare API token for cert-manager DNS-01 (`Zone:DNS:Edit` +
  `Zone:Zone:Read` on `itguys.ro`).
- Nextcloud admin password, MariaDB root/user passwords.

(If this grows, consider SOPS+age or Sealed Secrets — out of scope for now.)

## Apply order (filled in by the implementation plan)

TBD by the writing-plans step. High level: cert-manager → ClusterIssuer +
Certificate → namespace + secrets → MariaDB + Valkey → Nextcloud (Helm) →
NodePort + DNS → backup CronJob.

## Status

Design approved (brainstorming); see `docs/2026-05-17-nextcloud-k3s-design.md`.
Implementation plan not yet written. **No cluster changes made yet.**
