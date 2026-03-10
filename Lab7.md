# ArgoCD — Helm Repository & Application Deployment
**Kind Cluster | kubectl | GitHub | GitOps**  
*End-to-End Setup & Application Deployment*  
*February 2026*

---

## Table of Contents

1. [Connect a Helm Repository](#1-connect-a-helm-repository)
2. [Create an Application via the ArgoCD UI](#2-create-an-application-via-the-argocd-ui)
3. [Application Manifest (YAML)](#3-application-manifest-yaml)

---

## 1. Connect a Helm Repository

In the ArgoCD UI, navigate to **Settings → Repositories → Connect Repo**.

Fill in the connection details:

| Field | Value |
|-------|-------|
| **Connection Method** | HTTPS |
| **Type** | Helm |
| **Name** | `helmrepo` |
| **Project** | `myproj` |
| **Helm Chart Repository URL** | `https://charts.bitnami.com/bitnami` |

Click **Connect**. A green success indicator confirms the Helm repository is linked.

---

## 2. Create an Application via the ArgoCD UI

Navigate to **Applications → + NEW APP** and fill in the following sections:

### General

| Field | Value |
|-------|-------|
| **Application Name** | `myhelm` |
| **Project** | `myproj` |
| **Sync Policy** | Automated |
| **Auto-Create Namespace** | ✅ Enabled |

### Source

| Field | Value |
|-------|-------|
| **Repository URL** | `https://charts.bitnami.com/bitnami` |
| **Chart** | `nginx` |
| **Target Revision** | `22.4.3` |
| **Values Files** | `values.yaml` |

### Destination

| Field | Value |
|-------|-------|
| **Cluster** | `https://kubernetes.default.svc` |
| **Namespace** | nginx |

Click **Create**. ArgoCD will begin syncing the Helm chart from the Bitnami repository.

---

## 3. Application Manifest (YAML)

The equivalent ArgoCD `Application` resource in YAML:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myhelm
spec:
  destination:
    namespace: ''
    server: https://kubernetes.default.svc
  source:
    path: ''
    repoURL: https://charts.bitnami.com/bitnami
    targetRevision: 22.4.3
    chart: nginx
    helm:
      valueFiles:
        - values.yaml
  sources: []
  project: myproj
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
      enabled: true
    syncOptions:
      - CreateNamespace=true
```

### Sync Policy Explained

| Setting | Value | Description |
|---------|-------|-------------|
| **Automated** | `true` | ArgoCD automatically syncs when drift is detected |
| **Prune** | `false` | Resources removed from Git are **not** automatically deleted |
| **Self Heal** | `true` | ArgoCD corrects any manual changes made directly to the cluster |
| **CreateNamespace** | `true` | Target namespace is created automatically if it does not exist |
