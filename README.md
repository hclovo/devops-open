# PromptMinder — Helm Chart for ArgoCD

This repository contains the Helm chart for PromptMinder, designed to be deployed via ArgoCD.

## Repository Structure

```
├── prompt-minder-helm/           # Helm chart
│   ├── Chart.yaml
│   ├── values.yaml               # Default values
│   ├── values-local.yaml         # Local environment overrides
│   ├── values-production.yaml    # Production environment overrides
│   └── templates/
└── README.md
```

## Prerequisites

- Kubernetes cluster
- ArgoCD installed on the cluster
- Docker (for building images locally)

## ArgoCD Deployment

In ArgoCD UI, create an Application with the following settings:

| Field | Local | Production |
|-------|-------|------------|
| Repo URL | `https://github.com/hclovo/devops-open.git` | Same |
| Path | `prompt-minder-helm` | Same |
| Target Revision | `main` | `main` |
| Helm Value Files | `values.yaml`, `values-local.yaml` | `values.yaml`, `values-production.yaml` |
| Destination Namespace | `prompt-minder` | `prompt-minder-prod` |
| Sync Policy | Automated (Prune + Self-Heal) | Manual |

## Secrets

### Local Development

```bash
kubectl create namespace prompt-minder

kubectl create secret generic prompt-minder-helm \
  --namespace prompt-minder \
  --from-literal=DATABASE_URL='postgresql://user:password@host:5432/dbname' \
  --from-literal=CLERK_SECRET_KEY='sk_test_xxx' \
  --from-literal=NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY='pk_test_xxx' \
  --from-literal=AUTH_SECRET='your-auth-secret' \
  --from-literal=SUPABASE_URL='https://xxx.supabase.co' \
  --from-literal=SUPABASE_ANON_KEY='your-anon-key' \
  --from-literal=ZHIPU_API_KEY='your-zhipu-key'
```

### Production

Production uses `existingSecret: "prompt-minder-secrets"` to reference an externally managed secret (e.g. via External Secrets Operator or Sealed Secrets):

```bash
kubectl create namespace prompt-minder-prod

kubectl create secret generic prompt-minder-secrets \
  --namespace prompt-minder-prod \
  --from-literal=DATABASE_URL='postgresql://...' \
  --from-literal=CLERK_SECRET_KEY='sk_live_xxx' \
  --from-literal=NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY='pk_live_xxx' \
  --from-literal=AUTH_SECRET='...' \
  --from-literal=SUPABASE_URL='...' \
  --from-literal=SUPABASE_ANON_KEY='...' \
  --from-literal=ZHIPU_API_KEY='...'
```

## Key Design Decisions

### existingSecret

When `existingSecret` is set in values:
- The chart does **not** create a Secret resource
- The Deployment references the named secret via `secretRef`
- This allows secrets to be managed externally (External Secrets, Sealed Secrets, Vault, etc.)

When `existingSecret` is empty (default):
- The chart creates a Secret from `.Values.secrets`
- Useful for local development where secrets are passed via values files

### Database Migration

The `migration.enabled` flag controls an init container that runs `node drizzle/migrate.js` before the main application starts.

## Helm Template Preview

```bash
# Local
helm template prompt-minder ./prompt-minder-helm \
  -f prompt-minder-helm/values-local.yaml

# Production
helm template prompt-minder ./prompt-minder-helm \
  -f prompt-minder-helm/values-production.yaml
```