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
  namespace: argocd
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



```


## Part 6: Check Application Sync Status

```bash
# Get detailed app status
argocd app get hook

# View application in YAML
kubectl get application hook -n argocd -o yaml


```

---
