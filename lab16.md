# ArgoCD ApplicationSet — Matrix Generator Guide

> **Project:** `default` | **Cluster:** `https://kubernetes.default.svc` | **Environments:** `dev` · `qa` · `prod`

---

## 1. Overview

This ApplicationSet uses the **Matrix Generator** — a combination of two generators:

- **Git Directory Generator** — discovers application directories from the repository
- **List Generator** — defines a fixed set of target environments (`dev`, `qa`, `prod`)

The matrix produces the **cartesian product** of both generators. Every discovered app directory is deployed to every environment, resulting in one ArgoCD Application per `(app × environment)` pair.

**Example:** 3 app directories × 3 environments = **9 Applications** created automatically.

---

## 2. YAML Manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: argo-projects
  namespace: argocd
spec:
  generators:
  - matrix:
      generators:
      # Generator 1: discovers app directories from Git
      - git:
          repoURL: https://github.com/amitopenwriteup/ArgoCD-ApplicationSet-Demo.git
          revision: HEAD
          directories:
          - path: git-dir-generator-example/argo-projects/*
      # Generator 2: fixed list of target environments
      - list:
          elements:
          - namespace: dev
          - namespace: qa
          - namespace: prod
  template:
    metadata:
      # Unique name: <app>-<env> e.g. app-frontend-prod
      name: '{{path.basename}}-{{namespace}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/amitopenwriteup/ArgoCD-ApplicationSet-Demo.git
        targetRevision: HEAD
        # Manifests are read from the Git-discovered path
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        # Each app deploys into its environment namespace
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

---



## 4. Template Variables

| Variable | Source | Example Value |
|---|---|---|
| `{{path}}` | Git generator | `git-dir-generator-example/argo-projects/app-frontend` |
| `{{path.basename}}` | Git generator | `app-frontend` |
| `{{namespace}}` | List generator | `dev` / `qa` / `prod` |
| `{{path.basename}}-{{namespace}}` | Combined | `app-frontend-prod` |

> Variables from both generators are merged and available in the template simultaneously.

---

## 5. Field Reference

| Field | Value | Description |
|---|---|---|
| `generators[].matrix` | — | Activates the Matrix generator |
| `matrix.generators[0].git.repoURL` | HTTPS GitHub URL | Source repo for directory discovery |
| `matrix.generators[0].git.revision` | `HEAD` | Tracks latest commit |
| `matrix.generators[0].git.directories` | `argo-projects/*` | Glob to discover app directories |
| `matrix.generators[1].list.elements` | `dev`, `qa`, `prod` | Fixed environment list |
| `template.metadata.name` | `{{path.basename}}-{{namespace}}` | Unique Application name per pair |
| `spec.project` | `default` | ArgoCD project for RBAC scoping |
| `source.path` | `{{path}}` | Full path to Kubernetes manifests |
| `destination.server` | `https://kubernetes.default.svc` | In-cluster deployment target |
| `destination.namespace` | `{{namespace}}` | One namespace per environment |
| `automated.prune` | `true` | Removes resources deleted from Git |
| `automated.selfHeal` | `true` | Reverts manual cluster changes |
| `CreateNamespace=true` | syncOption | Auto-creates namespace if missing |

---



### Apply the Manifest

```bash
kubectl apply -f applicationset.yaml -n argocd
```

### Verify Applications Were Created

```bash
# List all generated Applications
argocd app list

# Filter by environment
argocd app list | grep "\-prod"

# Check via kubectl
kubectl get applications -n argocd

# Verify namespaces were created
kubectl get namespaces | grep -E "dev|qa|prod"
```

---

## 10. Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| Fewer apps than expected | Wrong directory glob | Verify path exists in repo with `argocd app get` |
| `namespace` variable empty | List element key mismatch | Ensure key is exactly `namespace:` in list elements |
| All apps point to same namespace | Variables not merging | Confirm matrix has exactly 2 child generators |
| `cluster not registered` error | Wrong destination server | Use `https://kubernetes.default.svc` for in-cluster |
| Namespace not auto-created | Missing syncOption | Add `CreateNamespace=true` to `syncOptions` |
| App name collision | Duplicate dir + env pair | Ensure `{{path.basename}}-{{namespace}}` is unique |

---

## 11. Matrix Generator Constraints

- The matrix generator accepts **exactly 2 child generators** — no more, no less
- Child generators are evaluated independently; their outputs are combined pairwise
- The total number of Applications = `(count of Git directories) × (count of list elements)`
- For more complex combinations, nest matrix generators or use the **Merge Generator**

---

*The Matrix Generator is ideal for multi-environment GitOps workflows where the same set of applications must be consistently deployed across dev, qa, and prod with a single source of truth.*
