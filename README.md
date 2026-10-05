# DevOpps — Architecture Documentation

## Overview

This repository holds the complete Infrastructure-as-Code (IaC) and GitOps configuration for deploying  full-stack applications on Kubernetes. Deployments are declarative, versioned in Git, and reconciled into the cluster by Argo CD. Container images are built by GitHub Actions and published to GHCR.

**Note:** HashiCorp Vault has been removed from this architecture. Secret management now uses plain Kubernetes `Secret` objects (with a migration path to SealedSecrets recommended).

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Full-Stack Architecture                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐     ┌──────────────────┐    ┌─────────────┐   │
│  │  GitHub      │     │  GitHub Actions  │    │  Build      │   │
│  │  Repository  │────▶│  (CI/CD)         │────▶│  Runners    │   │
│  └──────────────┘     └──────────────────┘    └─────────────┘   │
│         │                      │                     │          │
│         ▼                      ▼                     ▼          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │           GitHub Container Registry (GHCR)              │    │
│  │  ┌──────────────────────┐  ┌────────────────────────┐   │    │
│  │  │  Backend Images      │  │  Frontend Images       │   │    │
│  │  └──────────────────────┘  └────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│         │                      │                                │
│         ▼                      ▼                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │           Kubernetes Secret Objects                     │    │
│  │  ┌──────────────────────┐  ┌────────────────────────┐   │    │
│  │  │  Backend Secrets     │  │  Frontend Secrets      │   │    │
│  │  └──────────────────────┘  └────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│         │                      │                                │
│         ▼                      ▼                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    GitOps Repository                    │    │
│  │  ┌──────────────────────┐  ┌────────────────────────┐   │    │
│  │  │  Services/Backend/   │  │  Services/Frontend/    │   │    │
│  │  │  prod/deployment.yaml│  │  prod/deployment.yaml  │   │    │
│  │  └──────────────────────┘  └────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                      ArgoCD                             │    │
│  │  ┌─────────────────────────────────────────────────┐    │    │
│  │  │  Sync & Deploy                                  │    │    │
│  │  └─────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────┘    │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Kubernetes Cluster                         │    │
│  │  ┌─────────────────────────────────────────────────┐    │    │
│  │  │  Namespace: backend-prod                        │    │    │
│  │  │  ├── API Service                                │    │    │
│  │  │  ├── PostgreSQL Database                        │    │    │
│  │  │  └── Persistent Storage                         │    │    │
│  │  └─────────────────────────────────────────────────┘    │    │
│  │  ┌─────────────────────────────────────────────────┐    │    │
│  │  │  Namespace: frontend-prod                       │    │    │
│  │  │  ├── React/Vue/Angular App                      │    │    │
│  │  │  └── Static Assets                              │    │    │
│  │  └─────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

**Key components:**

- **Backend Service** — REST API + PostgreSQL
- **Frontend Service** — React/Vue/Angular SPA served by nginx
- **Argo CD** — GitOps reconciler, syncs Git state into the cluster
- **GHCR** — private image registry
- **GitHub Actions** — CI pipeline builds and pushes images, then updates the GitOps repo

---

## Prerequisites

**Tools:**
- `kubectl`, `argocd`, `helm`, `git`

**Access:**
- GitHub repo (write), GHCR, Kubernetes cluster (admin), Argo CD

---

## Repository Structure

```
DevOpps/
├── Services/
│   ├── Backend/
│   │   └── prod/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       ├── ingress.yaml
│   │       ├── configmap.yaml
│   │       ├── secrets.yaml      # plain Kubernetes Secret
│   │       ├── pvc.yaml
│   │       └── kustomization.yaml
│   └── Frontend/
│       └── prod/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── ingress.yaml
│           ├── configmap.yaml
│           ├── secrets.yaml
│           └── kustomization.yaml
├── ArgoCD/
│   ├── backend-application.yaml
│   └── frontend-application.yaml
├── Monitoring/
│   ├── prometheus-config.yaml
│   └── grafana-dashboards.yaml
└── README.md
```

---

## CI/CD Pipeline

Triggered on every push to `main`.

**Steps:**

1. Checkout code
2. Normalize repo name (lowercase)
3. Authenticate to GHCR
4. Build backend and frontend images
5. Push to GHCR with tags: `latest` and `{commit-hash}`
6. Update GitOps manifests with the new image tags
7. Commit and push to the GitOps repo (triggers Argo CD sync)

**Required GitHub Secrets:**

| Secret | Purpose |
|---|---|
| `GITHUB_TOKEN` | GHCR access |
| `GITOPS_PAT` | Push to GitOps repo |

---

## Deployment Guide

### Backend

```yaml
# Services/Backend/prod/deployment.yaml
image: ghcr.io/craygroup/backend:{NEW_COMMIT_HASH}
envFrom:
  - secretRef:
      name: backend-db-secrets
```

```yaml
# Services/Backend/prod/configmap.yaml
data:
  PORT: "3003"
  POSTGRES_HOST: "backend-postgres-service.backend-prod.svc.cluster.local"
```

### Frontend

```yaml
# Services/Frontend/prod/deployment.yaml
image: ghcr.io/craygroup/frontend:{NEW_COMMIT_HASH}
```

```yaml
# Services/Frontend/prod/configmap.yaml
data:
  VITE_BASE_URL: "https://api-backend.craygroup.biz"
  VITE_JINCE_URL: "https://qa-jince.craygroup.biz"
```

> **Reminder:** Vite env vars are baked in at build time. Changing `VITE_*` in a ConfigMap does nothing to a compiled SPA — you must rebuild the image with `--build-arg` and redeploy.

### Manual Deploy

```bash
kubectl apply -f Services/Backend/prod/
kubectl apply -f Services/Frontend/prod/

kubectl get pods -n backend-prod
kubectl get pods -n frontend-prod
```

### Rollback

```bash
argocd app rollback backend-service <revision-number>
argocd app rollback frontend-service <revision-number>

# or via Git
git revert <commit-hash> && git push origin main
```

---

## Secret Management (Vault Removed)

Secrets are now stored as **plain Kubernetes `Secret` objects** committed to the GitOps repo and reconciled by Argo CD.

### Secret Structure

**Backend (`Services/Backend/prod/secrets.yaml`):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-db-secrets
  namespace: backend-prod
type: Opaque
stringData:
  POSTGRES_USER: "user"
  POSTGRES_PASSWORD: "secure_password"
  POSTGRES_DB: "database"
  JWT_SECRET: "jwt_secret_key"
  API_KEY: "api_key"
```

**Frontend (`Services/Frontend/prod/secrets.yaml`):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: frontend-secrets
  namespace: frontend-prod
type: Opaque
stringData:
  VITE_API_KEY: "api_key"
  VITE_ANALYTICS_ID: "analytics_id"
  VITE_SENTRY_DSN: "sentry_dsn"
```

### Adding a New Secret

1. Edit the appropriate `secrets.yaml` with the new key.
2. Commit and push.
3. Argo CD syncs; pods pick up the change on next restart.
4. Restart the deployment to force a reload:

   ```bash
   kubectl rollout restart deployment/backend-app -n backend-prod
   ```

### Rotating Secrets

1. Update the value in `secrets.yaml`.
2. Commit and push.
3. Restart:

   ```bash
   kubectl rollout restart deployment -n backend-prod -l app=backend
   ```

> **Security note:** Plain `Secret` objects are only base64-encoded, not encrypted. Anyone with read access to the namespace can decode them. Recommended alternatives: **SealedSecrets** (Bitnami) or **SOPS + age** — both keep the Git repo safe while removing the Vault dependency.

---

## Monitoring

### Health Endpoints

**Backend:**
- `/health` — general health
- `/readiness` — readiness probe
- `/liveness` — liveness probe

**Frontend:**
- `/` — serves `index.html`, checked via HTTP probe

### Useful Commands

```bash
kubectl get pods -n backend-prod
kubectl get pods -n frontend-prod
kubectl logs -f -n backend-prod -l app=backend
kubectl logs -f -n frontend-prod -l app=frontend
kubectl get svc,ingress,pvc -n backend-prod
kubectl top pods -n backend-prod
kubectl top nodes
```

---

## Troubleshooting

### Pods Not Starting

```bash
kubectl describe pod -n backend-prod <pod-name>
kubectl get events -n backend-prod --sort-by='.lastTimestamp'
```

- `ImagePullBackOff` → check image name and GHCR credentials
- `CrashLoopBackOff` → check app logs and probes
- `Pending` → check resource limits and node capacity

### Database Connection Issues

```bash
kubectl run -it --rm postgres-test --image=postgres:15 -- bash
psql -h backend-postgres-service.backend-prod.svc.cluster.local -U postgres

kubectl logs -n backend-prod deployment/backend-postgres
kubectl get secrets -n backend-prod backend-db-secrets -o yaml
```

### Ingress Issues

```bash
kubectl get pods -n traefik
curl -v https://api-backend.craygroup.biz
curl -v https://frontend-app.craygroup.biz

kubectl get certificate -n backend-prod
kubectl describe certificate -n backend-prod backend-tls
```

### Argo CD Sync Issues

```bash
argocd app get backend-service
argocd app sync backend-service --force
argocd app refresh backend-service
```

### Secret Issues

```bash
kubectl get secrets -n backend-prod
kubectl describe secret -n backend-prod backend-db-secrets
kubectl get secret -n backend-prod backend-db-secrets -o jsonpath='{.data.POSTGRES_USER}' | base64 -d
```

### Collecting Logs

```bash
kubectl logs -n backend-prod deployment/backend-app > backend.log
kubectl logs -n frontend-prod deployment/frontend-app > frontend.log
kubectl logs -n backend-prod deployment/backend-postgres > postgres.log
kubectl get events -A > events.log
```

---

## Security Best Practices

**Secrets:**
- Do not commit raw secrets to Git if the repo is shared. Use SealedSecrets or SOPS.
- Rotate on a schedule.
- Enable audit logging on the cluster for `get secret` operations.
- Restrict `get secrets` via RBAC to only the service accounts that need it.

**Containers:**
- Pull from private GHCR only
- Minimal base images (`node:20-alpine`, `nginx:stable-alpine`)
- Image scanning in CI (Trivy/Grype)
- Run as non-root
- Read-only root filesystem where possible

**Network:**
- TLS terminated at Ingress
- Internal services use ClusterIP
- CORS restricted to known origins
- Rate limiting on API endpoints

**RBAC:**
- Least privilege
- Service accounts with minimal permissions
- Namespace isolation

---

## Scaling

```bash
# Horizontal
kubectl scale deployment -n backend-prod backend-app --replicas=5
kubectl autoscale deployment -n backend-prod backend-app --cpu-percent=70 --min=2 --max=10

# Vertical (edit deployment.yaml)
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "1"
```

**Database scaling:** read replicas, connection pooling, query/index optimization.

---

## Disaster Recovery

**Backup:**
- Database: daily full backups (Longhorn snapshot or `pg_dump` to object storage)
- Configuration: Git is the source of truth
- Secrets: encrypted backup of the `secrets.yaml` files
- Images: GHCR retains all pushed tags

**Restore:**

```bash
# Database from Longhorn
kubectl apply -f restore-pvc.yaml

# Or from pg_dump
kubectl exec -it -n backend-prod deployment/backend-postgres -- \
  pg_restore -U postgres -d postgres /backup/backup.dump

# Application rollback via ArgoCD
argocd app rollback backend-service <revision>

# Full cluster
kubectl apply -f Services/Backend/prod/
kubectl apply -f Services/Frontend/prod/
```

---

## Contributing

**Adding a new service:**

1. `mkdir -p Services/NewService/prod`
2. Create `deployment.yaml`, `service.yaml`, `ingress.yaml`, `configmap.yaml`, `secrets.yaml`
3. Add `ArgoCD/newservice-application.yaml` following the existing pattern
4. Update this README

**Updating secrets:**

1. Edit the relevant `secrets.yaml`
2. Commit and push
3. `kubectl rollout restart deployment -n <namespace> <app>`

**Testing locally:**

```bash
kubectl apply --dry-run=client -f Services/Backend/prod/
argocd app sync backend-service --dry-run
```

---

## Contacts

| Team | Contact |
|---|---|
| DevOps | princeben9312@gmail.com |
| On-Call | +254741414892 |

---

## License

Proprietary and confidential. Unauthorized copying or distribution prohibited.

**Maintainer:** Prince Benedict Wachira
**Version:** 1.0.0
**Last Updated:** Sun Oct 5, 2026
