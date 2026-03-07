# ArgoCD - Three Tier Cross Namespace Application Lab

---

## What We Will Deploy

```
┌──────────────────────────────────────────────────────────────┐
│              Three Tier Architecture                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Tier 1: Frontend   → namespace: frontend                   │
│  Tier 2: Backend    → namespace: backend                    │
│  Tier 3: Database   → namespace: database                   │
│                                                              │
│  All deployed from a single ArgoCD Application              │
│  pointing to: amitopenwriteup/threetieracrossns             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 1: Clone the Repository

```bash
cd ~
git clone https://github.com/amitopenwriteup/threetieracrossns.git
cd threetieracrossns

# View all manifests
ls -la
cat *.yaml
```

---

## Part 2: Create the ArgoCD Application

### File: application.yaml

```bash
vi application.yaml
```

Add the following content:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hook
spec:
  destination:
    namespace: ''
    server: https://kubernetes.default.svc
  source:
    path: .
    repoURL: https://github.com/amitopenwriteup/threetieracrossns.git
    targetRevision: HEAD
  sources: []
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      enabled: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
      - Validate=false
      - PruneLast=true
```

---

## Part 3: syncPolicy Options Explained

```
┌──────────────────────────────────────────────────────────────┐
│              syncPolicy Options Explained                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  automated:                                                  │
│    prune: true       → delete resources removed from Git    │
│    selfHeal: true    → auto fix any drift from desired state│
│    enabled: true     → automated sync is active             │
│                                                              │
│  syncOptions:                                                │
│    CreateNamespace=true   → create ns if not exists         │
│    ApplyOutOfSyncOnly=true → only sync changed resources    │
│    Validate=false         → skip kubectl validation         │
│    PruneLast=true         → delete old resources last       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 4: Apply the Application

```bash
# Apply to ArgoCD namespace
kubectl apply -f application.yaml -n argocd

# Verify application created
kubectl get application -n argocd
argocd app list
```

```
┌──────────────────────────────────────────────────────────────┐
│              argocd app list output                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  NAME  CLUSTER                    NAMESPACE  STATUS  HEALTH  │
│  hook  https://kubernetes.default.svc  ''  Synced  Healthy  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 5: Verify Deployment

```bash
# Check application status
argocd app get hook

# Check all namespaces created
kubectl get namespaces

# Check pods across all namespaces
kubectl get pods --all-namespaces

# Check services across all namespaces
kubectl get svc --all-namespaces

# Check all resources deployed
kubectl get all --all-namespaces | grep -v kube-system
```

```
┌──────────────────────────────────────────────────────────────┐
│              Expected Namespaces                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  kubectl get namespaces output:                              │
│                                                              │
│  NAME        STATUS   AGE                                    │
│  argocd      Active   ...                                    │
│  frontend    Active   ...  ← created by ArgoCD              │
│  backend     Active   ...  ← created by ArgoCD              │
│  database    Active   ...  ← created by ArgoCD              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Part 6: Check Application Sync Status

```bash
# Get detailed app status
argocd app get hook

# View application in YAML
kubectl get application hook -n argocd -o yaml

# Watch sync progress
argocd app sync hook --watch

# Check application history
argocd app history hook
```

---

## Part 7: Manual Sync (if needed)

```bash
# Trigger manual sync
argocd app sync hook

# Sync with force
argocd app sync hook --force

# Sync and prune
argocd app sync hook --prune
```

---

## Part 8: Delete the Application

```bash
# Delete application (keeps deployed resources)
argocd app delete hook

# Delete application and all deployed resources
argocd app delete hook --cascade

# Verify deletion
argocd app list
kubectl get namespaces
```

```
┌──────────────────────────────────────────────────────────────┐
│              delete vs delete --cascade                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  argocd app delete hook                                      │
│    → removes ArgoCD Application only                        │
│    → deployed resources remain in cluster                   │
│                                                              │
│  argocd app delete hook --cascade                            │
│    → removes ArgoCD Application                             │
│    → deletes ALL deployed resources from cluster ❌          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Command | Description |
|---|---|
| `kubectl apply -f application.yaml -n argocd` | Create ArgoCD Application |
| `argocd app list` | List all applications |
| `argocd app get hook` | Get application details |
| `argocd app sync hook` | Manually sync application |
| `argocd app sync hook --force` | Force sync |
| `argocd app history hook` | View sync history |
| `kubectl get application -n argocd` | List via kubectl |
| `kubectl get pods --all-namespaces` | Check all pods |
| `kubectl get namespaces` | Verify namespaces created |
| `argocd app delete hook` | Delete application only |
| `argocd app delete hook --cascade` | Delete app + all resources |

---

> **Note:** The `destination.namespace` is left empty (`''`) because this application deploys resources across multiple namespaces. Each manifest inside the repo defines its own namespace, and `CreateNamespace=true` ensures they are created automatically.
