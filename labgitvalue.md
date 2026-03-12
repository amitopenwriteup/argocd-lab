# helm-application-amit — ArgoCD ApplicationSet

A GitOps-driven ArgoCD **ApplicationSet** that deploys an NGINX Helm chart across multiple environments (`dev`, `qa`) using a **Matrix generator** strategy.

---

## Overview

| Field | Value |
|---|---|
| **ApplicationSet Name** | `helm-application-amit` |
| **Namespace** | `argocd` |
| **Chart** | `nginx` (Bitnami) |
| **Chart Version** | `22.4.3` |
| **Release Name** | `my-nginx` |
| **Git Repo** | [amitopenwriteup/nginx-proj](https://github.com/amitopenwriteup/nginx-proj.git) |

---

## How It Works

### Generator: Matrix

The **Matrix generator** combines two sub-generators to produce one ArgoCD `Application` per environment:

```
Matrix
├── List Generator     →  chart, releaseName, targetRevision
└── Git Generator      →  env, valuesFile  (from each matched values file)
```

#### 1. List Generator
Defines the static Helm chart parameters:

| Parameter | Value |
|---|---|
| `chart` | `nginx` |
| `releaseName` | `my-nginx` |
| `targetRevision` | `22.4.3` |

#### 2. Git Generator
Scans the Git repository for environment-specific values files:

| File Path | Derived `env` |
|---|---|
| `nginx-helm/dev/dev-values.yaml` | `dev` |
| `nginx-helm/qa/qa-values.yaml` | `qa` |

The `env` variable and `valuesFile` path are extracted automatically from each matched file.

---

## Generated Applications

The Matrix combination produces **two ArgoCD Applications**:

| Application Name | Namespace | Values File |
|---|---|---|
| `my-nginx-dev` | `amit-dev` | `nginx-helm/dev/dev-values.yaml` |
| `my-nginx-qa` | `amit-qa` | `nginx-helm/qa/qa-values.yaml` |

---

## Template

### Sources (Multi-Source)

This ApplicationSet uses **two sources** per Application:

| Source | Purpose |
|---|---|
| `https://charts.bitnami.com/bitnami` | Pulls the `nginx` Helm chart at version `22.4.3` |
| `https://github.com/amitopenwriteup/nginx-proj.git` | Provides the environment-specific `values.yaml` files via `ref: values` |

The Helm chart references the values file from the Git source using the `$values` reference alias:
```yaml
valueFiles:
  - $values/{{valuesFile}}
```

### Destination

Each application is deployed to the **local cluster** in a dynamically named namespace:

```
server:    https://kubernetes.default.svc
namespace: amit-{{env}}     →  amit-dev  /  amit-qa
```

### Sync Policy

| Setting | Value | Description |
|---|---|---|
| `automated.prune` | `true` | Removes resources no longer defined in Git |
| `automated.selfHeal` | `true` | Reverts manual changes back to the Git state |
| `CreateNamespace` | `true` | Auto-creates the target namespace if it doesn't exist |

---

## Repository Structure

```
nginx-proj/
└── nginx-helm/
    ├── dev/
    │   └── dev-values.yaml      # Dev environment overrides
    └── qa/
        └── qa-values.yaml       # QA environment overrides
```

---

## Prerequisites

- ArgoCD installed in the `argocd` namespace
- Access to the Bitnami Helm chart repository
- The Git repository `amitopenwriteup/nginx-proj` accessible from ArgoCD

---
```
vi applicationset.yaml

```
## ApplicationSet YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: helm-application-amit
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          - list:
              elements:
                - chart: nginx
                  releaseName: my-nginx
                  targetRevision: 22.4.3
          - git:
              repoURL: https://github.com/amitopenwriteup/nginx-proj.git
              revision: HEAD
              files:
                - path: nginx-helm/dev/dev-values.yaml
                - path: nginx-helm/qa/qa-values.yaml
  template:
    metadata:
      name: '{{releaseName}}-{{env}}'           # my-nginx-dev / my-nginx-qa
    spec:
      project: default
      sources:
        - repoURL: https://charts.bitnami.com/bitnami
          chart: '{{chart}}'
          targetRevision: '{{targetRevision}}'
          helm:
            releaseName: '{{releaseName}}'
            valueFiles:
              - $values/{{valuesFile}}
        - repoURL: https://github.com/amitopenwriteup/nginx-proj.git
          targetRevision: HEAD
          ref: values
      destination:
        server: https://kubernetes.default.svc
        namespace: 'amit-{{env}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```


## Deployment

Apply the ApplicationSet to your cluster:

```bash
kubectl apply -f applicationset.yaml -n argocd
```

Verify the generated Applications:

```bash
kubectl get applications -n argocd
# Expected output:
# my-nginx-dev   Synced   Healthy
# my-nginx-qa    Synced   Healthy
```

```bash
kubectl delete -f applicationset.yaml -n argocd
```
---
---

 namespace.
