# ArgoCD ApplicationSet — Git Directory Generator Guide

> **Project:** `stage-devseccops` | **Cluster:** `https://kubernetes.default.svc` | **Git:** HTTPS

---

## 1. Overview

This document describes the ArgoCD ApplicationSet resource using the **Git Directory Generator**. It automatically discovers directories in a GitHub repository and creates an ArgoCD Application for each one — enabling scalable, GitOps-driven multi-environment deployments.

**Key changes from the original configuration:**
- Repository URL changed from SSH (`git@github.com`) to **HTTPS** for broader compatibility
- Destination cluster changed to `https://kubernetes.default.svc` (in-cluster reference)

---

## 2. Updated YAML Manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: argo-projects
  namespace: argocd
spec:
  generators:
  - git:
      # HTTPS URL — no SSH key required
      repoURL: https://github.com/amitopenwriteup/ArgoCD-ApplicationSet-Demo.git
      revision: HEAD
      directories:
      - path: git-dir-generator-example/argo-projects/*
  template:
    metadata:
      # basename = directory name (e.g. app-frontend)
      name: '{{path.basename}}'
    spec:
      project: stage-devseccops
      source:
        repoURL: https://github.com/amitopenwriteup/ArgoCD-ApplicationSet-Demo.git
        targetRevision: HEAD
        # Full path to manifests directory
        path: '{{path}}'
      destination:
        # In-cluster endpoint — no external cluster registration needed
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

---

## 3. Field Reference

| Field | Description / Value |
|---|---|
| `apiVersion` | `argoproj.io/v1alpha1` — ArgoCD CRD API group |
| `kind` | `ApplicationSet` — manages a set of Argo Applications |
| `metadata.name` | `argo-projects` — name of this ApplicationSet |
| `metadata.namespace` | `argocd` — must match ArgoCD installation namespace |
| `generators[].git.repoURL` | HTTPS GitHub URL (no SSH key required) |
| `generators[].git.revision` | `HEAD` — tracks latest commit on default branch |
| `generators[].git.directories` | Glob path used to discover subdirectories |
| `template.metadata.name` | `{{path.basename}}` — Application name = directory name |
| `spec.project` | `stage-devseccops` — ArgoCD project scoping permissions |
| `source.repoURL` | Same HTTPS URL as the generator |
| `source.path` | `{{path}}` — full path to the manifests directory |
| `destination.server` | `https://kubernetes.default.svc` — in-cluster endpoint |
| `destination.namespace` | `{{path.basename}}` — one namespace per directory |
| `syncPolicy.automated.prune` | `true` — removes resources deleted from Git |
| `syncPolicy.automated.selfHeal` | `true` — reverts manual cluster changes |
| `CreateNamespace=true` | Auto-creates namespace if it does not exist |

---

## 4. SSH vs HTTPS Git URL

### Original (SSH)
```yaml
repoURL: git@github.com:amitopenwriteup/ArgoCD-ApplicationSet-Demo.git
```
Requires an SSH private key configured as a repository credential in ArgoCD. Suitable for private repos where SSH keys are managed.

### Updated (HTTPS)
```yaml
repoURL: https://github.com/amitopenwriteup/ArgoCD-ApplicationSet-Demo.git
```
Works with GitHub Personal Access Tokens (PAT) or no credentials at all for public repositories. Preferred for CI/CD pipelines and in-cluster deployments.

> **For private repos:** add credentials via:
> ```bash
> argocd repo add https://github.com/amitopenwriteup/ArgoCD-ApplicationSet-Demo.git \
>   --username <user> --password <PAT>
> ```

---

## 5. Destination: In-Cluster vs External

### External Cluster (original)
```yaml
server: https://92389502100E5959297F1C3CEC2CA298.sk1.ap-south-1.eks.amazonaws.com
```
Points to a remote Amazon EKS cluster. Requires that cluster to be registered with ArgoCD using its kubeconfig credentials.

### In-Cluster (updated)
```yaml
server: https://kubernetes.default.svc
```
The special value `https://kubernetes.default.svc` instructs ArgoCD to deploy to the **same cluster it is running in**. No additional cluster registration is required.

---

## 6. How the Git Directory Generator Works

The generator scans `git-dir-generator-example/argo-projects/*` and discovers all subdirectories. For each directory found, it creates one ArgoCD Application:

| Template Variable | Resolved Value |
|---|---|
| `{{path.basename}}` | Directory name, e.g. `app-frontend` |
| `{{path}}` | Full path, e.g. `git-dir-generator-example/argo-projects/app-frontend` |

**Example repository structure:**
```
git-dir-generator-example/argo-projects/
├── app-frontend/
│   ├── deployment.yaml
│   └── service.yaml
├── app-backend/
│   ├── deployment.yaml
│   └── service.yaml
└── app-database/
    ├── statefulset.yaml
    └── service.yaml
```

ArgoCD will automatically create **three Applications** — `app-frontend`, `app-backend`, and `app-database` — each syncing to its own namespace.

---

## 7. Sync Policy Behaviour

| Setting | Behaviour |
|---|---|
| `automated.prune: true` | Deletes cluster resources when removed from Git |
| `automated.selfHeal: true` | Reverts any manual `kubectl` changes to match Git state |
| `CreateNamespace=true` | Runs `kubectl create namespace` if it does not exist |

---

## 8. Prerequisites & Setup

- ArgoCD installed in the `argocd` namespace
- ArgoCD project `stage-devseccops` created with appropriate source/destination permissions
- For private repos: HTTPS credentials added (see Section 4)
- RBAC: ArgoCD service account needs permission to create namespaces

### Apply the Manifest

```bash
kubectl apply -f applicationset.yaml -n argocd
```

### Verify Applications Were Created

```bash
# List all ArgoCD Applications
argocd app list

# Check via kubectl
kubectl get applications -n argocd

# Watch sync status
argocd app list --output wide
```

### Check Namespaces

```bash
kubectl get namespaces | grep app-
```

---

## 9. Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| `PermissionDenied` on git clone | SSH URL with no key | Switch to HTTPS URL |
| `cluster not registered` error | External EKS endpoint used | Use `https://kubernetes.default.svc` |
| Namespace not created | `CreateNamespace=true` missing | Add to `syncOptions` |
| App not auto-synced | `automated` block missing | Add `syncPolicy.automated` |
| Directory not discovered | Wrong glob path | Verify path exists in repo |

---

*Generated configuration uses `https://kubernetes.default.svc` for in-cluster deployments and HTTPS GitHub URLs for credential-free access to public repositories.*
