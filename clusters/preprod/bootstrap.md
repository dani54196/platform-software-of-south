# Preprod Cluster Bootstrap

**Cluster:** preprod  
**Nodes:** 2× VM — 4 CPU / 8 GB RAM  
**Distribution:** k3s (1 master + 1 worker)  
**ArgoCD access:** `http://<MASTER_IP>:30080`  
**TLS:** None (internal access only)

---

## Prerequisites

- [ ] k3s installed and nodes joined
- [ ] `kubectl` configured with preprod kubeconfig
- [ ] `helm` v3 installed
- [ ] Access to this repo (cloned locally)
- [ ] GitHub PAT created with read access to this repo

Verify your context before running anything:

```bash
kubectl config current-context
kubectl get nodes
```

Expected output: 2 nodes, both `Ready`.

---

## Step 1 — Install ArgoCD

```bash
# Add Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Create namespace
kubectl create namespace argocd

# Install ArgoCD
helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 7.x.x \        # pin to latest stable: helm search repo argo/argo-cd
  -f bootstrap/argocd/values-preprod.yaml \
  --wait
```

Verify all pods are running:

```bash
kubectl get pods -n argocd
```

---

## Step 2 — Get the initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Access the UI at `http://<MASTER_IP>:30080`  
Login: `admin` / `<password from above>`

> **Change the password immediately** after first login:  
> UI → User Info → Update Password

---

## Step 3 — Register the private repo

Edit `bootstrap/argocd/repo-secret.yaml` with your actual GitHub credentials, then apply:

```bash
kubectl apply -f bootstrap/argocd/repo-secret.yaml
```

Verify in ArgoCD UI: Settings → Repositories — status should be `Successful`.

> Delete or do not commit `repo-secret.yaml` with real credentials.  
> See `docs/sealed-secrets.md` to encrypt secrets properly.

---

## Step 4 — Deploy the Root App

The root app is the single entry point that tells ArgoCD about every other app.

```bash
helm install root-app ./bootstrap/root-app \
  --namespace argocd \
  -f clusters/preprod/root-app-values.yaml
```

After a few seconds, open the ArgoCD UI — you should see all child applications appear and start syncing.

Force a manual sync if needed:

```bash
kubectl -n argocd exec -it deploy/argocd-server -- \
  argocd app sync root-app --insecure --server localhost:8080
```

---

## Step 5 — Verify

```bash
# All ArgoCD apps should be Synced + Healthy
kubectl get applications -n argocd

# Namespaces created by ArgoCD
kubectl get namespaces
```

---

## Accessing ArgoCD CLI (optional)

```bash
# Install argocd CLI
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

# Login
argocd login <MASTER_IP>:30080 --username admin --password <password> --insecure
```

---

## Troubleshooting

**Pod stuck in Pending:**
```bash
kubectl describe pod <pod-name> -n argocd
# Usually: insufficient resources or PVC issue
```

**App not syncing:**
```bash
kubectl logs -n argocd deploy/argocd-repo-server
# Check repo credentials and network access to GitHub
```

**Reset admin password:**
```bash
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {"admin.password": "", "admin.passwordMtime": ""}}'
kubectl rollout restart deploy/argocd-server -n argocd
# New password will be regenerated in argocd-initial-admin-secret
```