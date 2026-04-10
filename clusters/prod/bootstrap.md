# Prod Cluster Bootstrap

**Cluster:** prod  
**Node:** 1× OCI VM  
**Distribution:** k3s  
**ArgoCD access:** `https://argocd.softwareofsouth.website`  
**TLS:** Let's Encrypt via cert-manager + Traefik

---

## Prerequisites

- [ ] k3s installed on OCI VM
- [ ] `kubectl` configured with prod kubeconfig
- [ ] `helm` v3 installed
- [ ] Traefik running and accessible (port 443 open in OCI Security List)
- [ ] cert-manager installed with a `ClusterIssuer` named `letsencrypt-prod`
- [ ] DNS A record: `argocd.softwareofsouth.website` → OCI VM public IP
- [ ] GitHub PAT created with read access to this repo

Verify before proceeding:

```bash
kubectl config current-context
kubectl get nodes
kubectl get clusterissuer letsencrypt-prod
```

---

## Step 1 — Install ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

kubectl create namespace argocd

helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 7.x.x \        # pin version: helm search repo argo/argo-cd
  -f bootstrap/argocd/values-prod.yaml \
  --wait
```

Verify:

```bash
kubectl get pods -n argocd
```

---

## Step 2 — Expose ArgoCD via Traefik

Apply the IngressRoute and Certificate:

```bash
kubectl apply -f bootstrap/argocd/traefik-ingressroute.yaml
```

Wait for the TLS certificate to be issued (usually 1-2 minutes):

```bash
kubectl get certificate -n argocd argocd-tls -w
# Wait for: READY = True
```

Then access: `https://argocd.softwareofsouth.website`

---

## Step 3 — Get initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Login: `admin` / `<password from above>`

> **Change the password immediately** after first login.

---

## Step 4 — Register the private repo

```bash
# Edit with real credentials first (do not commit with real values)
kubectl apply -f bootstrap/argocd/repo-secret.yaml
```

Verify in UI: Settings → Repositories → status `Successful`.

---

## Step 5 — Deploy the Root App

```bash
helm install root-app ./bootstrap/root-app \
  --namespace argocd \
  -f clusters/prod/root-app-values.yaml
```

All child apps will appear in the ArgoCD UI and begin syncing automatically.

---

## Step 6 — Verify

```bash
kubectl get applications -n argocd
kubectl get namespaces
kubectl get certificate -n argocd
```

---

## OCI-specific: Open required ports

In OCI Console → Networking → Security Lists, ensure these ingress rules exist:

| Protocol | Port | Source     | Purpose         |
|----------|------|------------|-----------------|
| TCP      | 80   | 0.0.0.0/0  | HTTP (redirect) |
| TCP      | 443  | 0.0.0.0/0  | HTTPS / ArgoCD  |
| TCP      | 6443 | your IP/32 | kubectl API     |

---

## Troubleshooting

**Certificate stuck in Pending:**
```bash
kubectl describe certificate argocd-tls -n argocd
kubectl describe certificaterequest -n argocd
# Check: DNS resolves correctly and port 80 is open (ACME HTTP-01 challenge)
```

**Traefik not routing:**
```bash
kubectl get ingressroute -n argocd
kubectl logs -n kube-system deploy/traefik
```

**ArgoCD UI returns 502:**
```bash
# Confirm argocd-server is running insecure (check values-prod.yaml)
kubectl get pods -n argocd
kubectl logs -n argocd deploy/argocd-server
```