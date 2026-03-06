# ArgoCD — Lab 9: Deploy a Helm Chart via CLI
**Kind Cluster | kubectl | GitHub | GitOps**  
*End-to-End Setup & Application Deployment*  
*February 2026*

---

## Table of Contents

1. [Step 1 — Create the Helm Application](#step-1--create-the-helm-application)
2. [Step 2 — Verify & Inspect](#step-2--verify--inspect)
3. [Step 3 — Delete the Application](#step-3--delete-the-application)

---

## Step 1 — Create the Helm Application

Deploy the Bitnami nginx Helm chart using the ArgoCD CLI:

```bash
argocd app create myhelm-proj \
  --repo https://charts.bitnami.com/bitnami \
  --helm-chart nginx \
  --revision 22.4.3 \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --project myproj \
  --sync-policy automated \
  --self-heal \
  --sync-option CreateNamespace=true \
  --values values.yaml
```

### Flags Explained

| Flag | Description |
|------|-------------|
| `--repo` | Helm chart repository URL |
| `--helm-chart` | Name of the chart to deploy |
| `--revision` | Specific chart version to pin (`22.4.3`) |
| `--dest-server` | Target Kubernetes cluster API endpoint |
| `--dest-namespace` | Namespace to deploy the chart into |
| `--project` | ArgoCD project to assign the application to |
| `--sync-policy automated` | ArgoCD automatically syncs on detected drift |
| `--self-heal` | Reverts manual cluster changes back to desired state |
| `--sync-option CreateNamespace=true` | Creates the namespace if it does not exist |
| `--values` | Override values file to apply during chart rendering |

---

## Step 2 — Verify & Inspect

**List all applications:**

```bash
argocd app list
```

**Describe the application in detail:**

```bash
kubectl describe app myhelm-proj -n argocd
```

> ℹ️ The `kubectl describe` output will show sync status, health, conditions, and recent events for the application.

---

## Step 3 — Delete the Application

When finished, remove the application from ArgoCD:

```bash
argocd app delete myhelm-proj
```

> ⚠️ **Note:** To also remove all Kubernetes resources deployed by the app, add the `--cascade` flag:
> ```bash
> argocd app delete myhelm-proj --cascade
> ```
