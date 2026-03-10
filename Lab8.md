# ArgoCD — CLI: Register Repo & Deploy Application
**Kind Cluster | kubectl | GitHub | GitOps**  
*End-to-End Setup & Application Deployment*  
*February 2026*

---

## Table of Contents

1. [Register a GitHub Repository](#1-register-a-github-repository)
2. [Create an Application via CLI](#2-create-an-application-via-cli)
3. [Verify the Deployment](#3-verify-the-deployment)
4. [Delete the Application](#4-delete-the-application)

---


---

## 2. Create an Application via CLI

Deploy the nginx application from your GitHub repo to the cluster:

```bash
argocd app create myappcli \
  --repo https://github.com/amitopenwriteup/nginx-argoproj.git \
  --path nginx-conf \
  --dest-server https://kubernetes.default.svc \
  --project myproj \
  --revision main \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### Flags Explained

| Flag | Description |
|------|-------------|
| `--repo` | GitHub repository URL containing the manifests |
| `--path` | Path within the repo where manifests are located |
| `--dest-server` | Target Kubernetes cluster API endpoint |
| `--project` | ArgoCD project to assign the application to |
| `--revision` | Branch or tag to track (`main`) |
| `--sync-policy automated` | ArgoCD automatically syncs on detected drift |
| `--auto-prune` | Removes resources deleted from Git |
| `--self-heal` | Reverts manual cluster changes back to Git state |

---

## 3. Verify the Deployment

**List all ArgoCD applications:**

```bash
argocd app list
```

**Check namespaces on the cluster:**

```bash
kubectl get ns
```

**Verify pods are running in the deployed namespace:**

```bash
kubectl get pods -n stage-nginx-namespace
```

---

## 4. Delete the Application

When finished, clean up by deleting the application:

```bash
argocd app delete myappcli
```

> ℹ️ **Note:** Deleting the application in ArgoCD removes it from ArgoCD's control but does not automatically delete the deployed Kubernetes resources unless `--cascade` is specified:
> ```bash
> argocd app delete myappcli --cascade
> ```
