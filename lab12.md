# ArgoCD ApplicationSet - List Generator Lab

---

## What is ApplicationSet?
ApplicationSet automates the creation of multiple ArgoCD Applications from a single template using generators.
The **List Generator** creates one application per element defined in the list.

```
┌──────────────────────────────────────────────────────────────┐
│              List Generator Flow                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ApplicationSet (1 file)                                     │
│         │                                                    │
│         ├── element: namespace: dev  → dev-color-app        │
│         ├── element: namespace: test → test-color-app       │
│         └── element: namespace: uat  → uat-color-app        │
│                                                              │
│  3 elements → 3 Applications created automatically          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Step 1: Clone the Repository

```bash
git clone https://github.com/amitopenwriteup/argoappsetlist.git
cd argoappsetlist
```

---

## Step 2: View the ApplicationSet File

```bash
cat applicationset.yaml
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: color-applicaitonset
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - namespace: dev
      - namespace: test
      - namespace: uat
  template:
    metadata:
      # generates: dev-color-app, test-color-app, uat-color-app
      name: '{{namespace}}-color-app'
    spec:
      # applications will be in myproj project
      project: default
      source:
        repoURL: https://github.com/amitopenwriteup/argoappsetlist.git
        targetRevision: HEAD
        path: list-generator-example/deployment
      destination:
        # default cluster as destination
        server: https://kubernetes.default.svc
        # deploys to different namespaces
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

```
┌──────────────────────────────────────────────────────────────┐
│              ApplicationSet YAML Explained                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  generators.list.elements  → list of namespaces             │
│  template.metadata.name    → {{namespace}}-color-app        │
│  project                   → myproj                         │
│  source.repoURL            → git repo with manifests        │
│  source.path               → path inside repo               │
│  destination.namespace     → {{namespace}} (dynamic)        │
│  syncPolicy.automated      → auto sync enabled              │
│  prune: true               → delete removed resources       │
│  selfHeal: true            → auto fix drift                 │
│  CreateNamespace=true      → create ns if not exists        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Step 3: Apply the ApplicationSet

```bash
kubectl apply -f applicationset.yaml
```

```
┌──────────────────────────────────────────────────────────────┐
│              kubectl apply output                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  applicationset.argoproj.io/color-applicaitonset created    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Step 4: Verify ApplicationSet

```bash
# Full form
kubectl get applicationset -n argocd

# Short form
kubectl get appset -n argocd

# Using ArgoCD CLI
argocd appset list
```

```
┌──────────────────────────────────────────────────────────────┐
│              kubectl get appset output                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  NAME                  AGE                                   │
│  color-applicaitonset  30s                                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Step 5: Verify Applications Created

```bash
argocd app list
```
```
Go in Argocd UI and delete one application resource "dev-color-app"
```
```
┌──────────────────────────────────────────────────────────────┐
│              argocd app list output                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  NAME            CLUSTER                   NAMESPACE  STATUS │
│  dev-color-app   https://kubernetes.default.svc  dev   Synced│
│  test-color-app  https://kubernetes.default.svc  test  Synced│
│  uat-color-app   https://kubernetes.default.svc  uat   Synced│
│                                                              │
│  3 apps created automatically from 1 ApplicationSet ✅       │
└──────────────────────────────────────────────────────────────┘
```

---

## Step 6: Delete the ApplicationSet

```bash
kubectl delete -f applicationset.yaml
```

```
┌──────────────────────────────────────────────────────────────┐
│              After Delete                                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ApplicationSet deleted → all 3 apps deleted automatically  │
│                                                              │
│  dev-color-app   ❌ deleted                                  │
│  test-color-app  ❌ deleted                                  │
│  uat-color-app   ❌ deleted                                  │
│                                                              │
│  Verify:                                                     │
│  argocd app list        → no apps listed                    │
│  kubectl get appset -n argocd → no applicationset listed    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Command | Description |
|---|---|
| `kubectl apply -f applicationset.yaml` | Create ApplicationSet |
| `kubectl get applicationset -n argocd` | List ApplicationSets (full) |
| `kubectl get appset -n argocd` | List ApplicationSets (short) |
| `argocd appset list` | List via ArgoCD CLI |
| `argocd app list` | List all generated apps |
| `kubectl delete -f applicationset.yaml` | Delete ApplicationSet + apps |
