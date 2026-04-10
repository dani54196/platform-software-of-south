# ArgoCD Runbook

Quick reference for day-to-day ArgoCD operations.

---

## Architecture

```
clusters/preprod/root-app-values.yaml  ──▶  bootstrap/root-app/  ──▶  apps/*
clusters/prod/root-app-values.yaml     ──▶  bootstrap/root-app/  ──▶  apps/*
```

Each cluster runs its own ArgoCD instance and only manages itself.  
The root app is a Helm chart — adding a new app means adding a template to `bootstrap/root-app/templates/`.

---

## Common Commands

### App status

```bash
kubectl get applications -n argocd

# Detailed status
kubectl describe application <app-name> -n argocd
```

### Force sync

```bash
# Via CLI
argocd app sync <app-name>

# Via kubectl (triggers reconcile)
kubectl annotate application <app-name> -n argocd \
  argocd.argoproj.io/refresh=hard --overwrite
```

### Diff before sync

```bash
argocd app diff <app-name>
```

### Rollback to previous version

```bash
argocd app history <app-name>
argocd app rollback <app-name> <revision-id>
```

---

## Adding a New Application

1. Create the Helm chart under `apps/your-app/`
2. Add `values-preprod.yaml` and `values-prod.yaml` with environment-specific overrides
3. Add an ArgoCD `Application` manifest to `bootstrap/root-app/templates/your-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: your-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: default
  source:
    repoURL: {{ .Values.repoURL }}
    targetRevision: {{ .Values.targetRevision }}
    path: apps/your-app
    helm:
      valueFiles:
        - values.yaml
        - values-{{ .Values.environment }}.yaml
  destination:
    server: {{ .Values.destination.server }}
    namespace: your-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

4. Commit and push → ArgoCD will automatically detect and deploy.

---

## Sync Waves

Sync waves control deployment order. Lower numbers deploy first.

| Wave | What deploys |
|------|-------------|
| `0`  | Infrastructure (cert-manager, secrets) |
| `1`  | Monitoring stack |
| `2`  | Applications |

Set via annotation: `argocd.argoproj.io/sync-wave: "N"`

---

## Secrets Management

**Never commit plain secrets to git.**

Current approach: apply `repo-secret.yaml` manually, then delete it locally.

Next step: implement Sealed Secrets (see `docs/sealed-secrets.md`).

```bash
# Verify no secrets are tracked
git log --all --full-history -- "*secret*"
git log --all --full-history -- "*.env"
```

---

## Disaster Recovery

### ArgoCD itself went down

ArgoCD is stateless — all state is in the git repo.

```bash
# Reinstall ArgoCD
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  -f bootstrap/argocd/values-<env>.yaml \
  --wait

# Re-apply repo secret
kubectl apply -f bootstrap/argocd/repo-secret.yaml

# Re-deploy root app
helm upgrade --install root-app ./bootstrap/root-app \
  --namespace argocd \
  -f clusters/<env>/root-app-values.yaml
```

ArgoCD will re-sync everything from git automatically.

### Application is broken — disable auto-sync temporarily

```bash
argocd app set <app-name> --sync-policy none
# Fix the issue, then re-enable
argocd app set <app-name> --sync-policy automated
```