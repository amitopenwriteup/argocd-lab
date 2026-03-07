# ArgoCD Hooks — Application Deployment Guide

> **App:** `argocdhooks` | **Namespace:** `default` | **Cluster:** `https://kubernetes.default.svc` | **Repo:** `github.com/amitopenwriteup/argocdhooks`

---

## 1. Overview

This ArgoCD `Application` deploys Kubernetes manifests from the `argocdhooks` repository to an in-cluster destination. The repository contains a single `manifest.yaml` at the root (`path: .`) that includes **resource hooks** — annotated Kubernetes Jobs that run at specific phases of the ArgoCD sync lifecycle.

**What this configuration does:**

- Watches the `main` branch of the `argocdhooks` GitHub repository
- Automatically syncs any changes with `prune` and `selfHeal` enabled
- Executes hook Jobs in defined sync phases (PreSync, PostSync, SyncFail)
- Deploys all resources into the `default` namespace of the in-cluster Kubernetes API

---

## 2. ArgoCD Application Manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocdhooks
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/amitopenwriteup/argocdhooks.git
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 3. Field Reference

| Field | Value | Description |
|---|---|---|
| `metadata.name` | `argocdhooks` | ArgoCD Application name |
| `metadata.namespace` | `argocd` | Must be in the ArgoCD control plane namespace |
| `spec.project` | `default` | ArgoCD project for RBAC scoping |
| `source.repoURL` | `https://github.com/amitopenwriteup/argocdhooks.git` | HTTPS GitHub repo — no SSH key required |
| `source.targetRevision` | `HEAD` | Tracks the latest commit on the default branch |
| `source.path` | `.` | Root of the repository — all YAML files are applied |
| `destination.server` | `https://kubernetes.default.svc` | In-cluster Kubernetes API endpoint |
| `destination.namespace` | `default` | All resources deploy to the `default` namespace |
| `automated.prune` | `true` | Deletes resources removed from Git |
| `automated.selfHeal` | `true` | Reverts any manual `kubectl` changes |
| `CreateNamespace=true` | syncOption | Creates `default` namespace if missing (safe no-op if it exists) |

---

## 4. Repository Structure

```
argocdhooks/
└── manifest.yaml    ← single file containing all resources + hooks
```

Since `path: .` points to the repo root, ArgoCD reads **all YAML files** in the directory. The `manifest.yaml` contains a multi-document YAML file with Kubernetes resources and hook-annotated Jobs separated by `---`.

---

## 5. What Are ArgoCD Hooks?

Hooks are standard Kubernetes resources (typically `Job`) annotated with `argocd.argoproj.io/hook` to run at specific sync lifecycle phases.

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

### Sync Lifecycle with Hooks

```
Git push / sync triggered
         │
         ▼
  ┌─────────────┐
  │  PreSync    │  ← Hook Jobs run BEFORE manifests are applied
  │  (hooks)    │    e.g. DB migration, pre-flight checks
  └──────┬──────┘
         │ ✓ success
         ▼
  ┌─────────────┐
  │    Sync     │  ← Regular manifests applied (Deployments, Services...)
  │ (manifests) │    Sync-phase hooks run in parallel
  └──────┬──────┘
         │ ✓ all resources healthy
         ▼
  ┌─────────────┐
  │  PostSync   │  ← Hook Jobs run AFTER everything is healthy
  │  (hooks)    │    e.g. smoke tests, Slack notifications
  └──────┬──────┘
         │ ✗ failure at any phase
         ▼
  ┌─────────────┐
  │  SyncFail   │  ← Runs ONLY on failure
  │  (hooks)    │    e.g. alert, rollback trigger
  └─────────────┘
```

---

## 6. Typical `manifest.yaml` Structure

A hooks-based `manifest.yaml` in the root of the repo typically contains multiple YAML documents:

```yaml
# ── PreSync: runs before manifests are applied ──────────────────────────
apiVersion: batch/v1
kind: Job
metadata:
  generateName: pre-sync-hook-
  namespace: default
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: pre-sync
          image: alpine:3.18
          command: ["sh", "-c", "echo PreSync hook running && sleep 5"]
      restartPolicy: Never
  backoffLimit: 1
---
# ── Main Application Resource ────────────────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
  namespace: default
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80
---
# ── PostSync: runs after all resources are healthy ───────────────────────
apiVersion: batch/v1
kind: Job
metadata:
  generateName: post-sync-hook-
  namespace: default
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: post-sync
          image: curlimages/curl:latest
          command: ["sh", "-c", "curl -sf http://my-app-svc/healthz && echo PostSync passed"]
      restartPolicy: Never
  backoffLimit: 1
---
# ── SyncFail: runs only when sync fails ──────────────────────────────────
apiVersion: batch/v1
kind: Job
metadata:
  generateName: sync-fail-hook-
  namespace: default
  annotations:
    argocd.argoproj.io/hook: SyncFail
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: sync-fail
          image: alpine:3.18
          command: ["sh", "-c", "echo SYNC FAILED - sending alert"]
      restartPolicy: Never
  backoffLimit: 1
```

---

## 7. Hook Types Reference

| Hook | Runs When | Common Use |
|---|---|---|
| `PreSync` | Before any manifests are applied | DB migrations, pre-flight validation |
| `Sync` | Alongside manifest apply | Parallel setup tasks |
| `PostSync` | After all resources are Healthy | Health checks, smoke tests, notifications |
| `SyncFail` | When sync operation fails | Failure alerts, rollback triggers |
| `Skip` | Never | Exclude a resource from sync entirely |

---

## 8. Hook Delete Policies

| Policy | Behaviour |
|---|---|
| `HookSucceeded` | Auto-deletes Job after successful completion |
| `HookFailed` | Auto-deletes Job after failure |
| `BeforeHookCreation` | Deletes previous hook resource before creating a new one |

**Always-cleanup pattern:**

```yaml
argocd.argoproj.io/hook-delete-policy: HookSucceeded,HookFailed
```

---

## 9. Deploying the Application

### Apply the Application to ArgoCD

```bash
kubectl apply -f application.yaml -n argocd
```

### Trigger a Manual Sync

```bash
argocd app sync argocdhooks
```

### Watch Sync Progress

```bash
# Live sync status
argocd app get argocdhooks

# Watch hook Jobs execute
kubectl get jobs -n default -w

# View PreSync hook logs
kubectl logs -l argocd.argoproj.io/hook=PreSync -n default

# View PostSync hook logs
kubectl logs -l argocd.argoproj.io/hook=PostSync -n default
```

### Check Application Health

```bash
# Full app status
argocd app get argocdhooks --output wide

# Sync history
argocd app history argocdhooks

# All resources deployed
kubectl get all -n default
```

---

## 10. Sync Policy Behaviour

| Setting | Behaviour |
|---|---|
| `automated.prune: true` | If a resource is deleted from `manifest.yaml` in Git, it is deleted from the cluster on next sync |
| `automated.selfHeal: true` | If someone manually edits a resource in the cluster, ArgoCD reverts it to match Git within minutes |
| `CreateNamespace=true` | Ensures `default` namespace exists before applying resources (safe no-op if it already exists) |

---

## 11. Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| Hook Job stays `Pending` | Resource limits on cluster | `kubectl describe pod <hook-pod> -n default` |
| PreSync fails, sync aborted | Hook Job exit code non-zero | Check logs: `kubectl logs <pod> -n default` |
| PostSync not triggered | App resources not reaching Healthy | `argocd app get argocdhooks` — check health status |
| Old hook Job conflicts | Using `name:` without `BeforeHookCreation` | Use `generateName:` or add `BeforeHookCreation` delete policy |
| Resources re-appear after deletion | `prune: true` not set | Ensure `syncPolicy.automated.prune: true` |
| Manual changes reverted | `selfHeal: true` is active | Expected behaviour — all changes must go via Git |
| App stuck `OutOfSync` | `path: .` picking up non-YAML files | Confirm only `.yaml`/`.yml` files in repo root |

---

## 12. Quick Reference — Hook Annotations

```yaml
metadata:
  annotations:
    # Required: defines the hook phase
    argocd.argoproj.io/hook: PreSync

    # Optional: controls cleanup after execution
    argocd.argoproj.io/hook-delete-policy: HookSucceeded

    # Optional: ordering within a phase (lower runs first)
    argocd.argoproj.io/sync-wave: "1"
```

---

*This Application follows a GitOps-first pattern — `manifest.yaml` is the single source of truth. All changes to hooks, deployments, and services flow through Git commits and are automatically reconciled by ArgoCD.*
