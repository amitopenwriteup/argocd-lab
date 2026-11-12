# Lab 2: ArgoCD Setup

*ArgoCD Workshop | kubectl | GitOps | February 2026*

**Goal:** Install ArgoCD on your Kind cluster, expose the UI, and log in as `admin`.

**Estimated time:** 15 minutes

**Prerequisite:** Lab 1 completed. The `argocd-workshop` Kind cluster is running and your current context is `kind-argocd-workshop`.

## 1. Confirm Your Cluster

```bash
kubectl config current-context
kubectl get nodes
```

The context should be `kind-argocd-workshop` and all nodes should be `Ready`. If not, go back to Lab 1.

## 2. Install ArgoCD

ArgoCD is installed into its own dedicated namespace using the official manifest from the ArgoCD project.

### 2.1 Create the namespace

```bash
kubectl create namespace argocd
```

### 2.2 Apply the install manifest

```bash
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2.3 Wait for all pods to be Running

```bash
kubectl -n argocd get pods
```

**Note:** It can take 2 to 3 minutes for all ArgoCD pods (`server`, `application-controller`, `repo-server`, `dex`, `redis`) to reach `Running`. Do not proceed until they are all Running. To watch progress, add `-w` and press Ctrl+C when done:

```bash
kubectl -n argocd get pods -w
```

## 3. Access the ArgoCD Server

### 3.1 Port-forward the ArgoCD server

Because Kind is a local cluster, port-forward the ArgoCD service to expose the UI on your machine:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 \
  --address 0.0.0.0 > argocd-portforward.log 2>&1 &
```

The `&` runs it in the background. If you prefer to run it in the foreground, remove the `> ... &` part and leave that terminal open. Open a **new terminal** for the remaining commands.

### 3.2 Open the ArgoCD UI

On the same machine, browse to `https://localhost:8080`. If you are connecting from another machine (for example a workshop VM), get the host IP first:

```bash
hostname -i
```

Then browse to `https://<your-ip>:8080`.

Your browser will warn about a self-signed certificate. This is expected for a local lab. Choose **Advanced**, then **Proceed**.

## 4. Retrieve the Admin Password

ArgoCD generates an initial admin password and stores it in a Kubernetes secret. Decode it with:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

**Login credentials**

- **Username:** `admin`
- **Password:** the output of the command above (copy it before logging in)

## 5. Log In

Enter `admin` as the username, paste the decoded password, and click **Sign In**.

You are now inside the ArgoCD dashboard.

## 6. Checkpoint

- [ ] All pods in the `argocd` namespace are `Running`
- [ ] The ArgoCD UI loads in your browser
- [ ] You are logged in as `admin`

## Troubleshooting

| Problem | Fix |
| --- | --- |
| Pods stuck in `Pending` or `ContainerCreating` | Wait a few minutes, then check with `kubectl -n argocd describe pod <pod-name>` |
| UI does not load | Check the port-forward is still running: `ps aux \| grep port-forward`. Restart it if needed |
| Port 8080 already in use | Use another local port, for example `8081:443`, and browse to that port |
| `argocd-initial-admin-secret` not found | The server may not be ready yet. Wait and retry |

## Cleanup (end of workshop only)

```bash
pkill -f "port-forward svc/argocd-server"
kubectl delete namespace argocd
```
