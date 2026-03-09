# ArgoCD High Availability Lab

## Overview

This lab guides you through deploying ArgoCD in a High Availability (HA) configuration on Kubernetes. The HA setup ensures redundancy and resilience for your GitOps continuous delivery platform.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Kubernetes Cluster | v1.25+ with at least 3 nodes |
| kubectl | Configured with cluster admin access |
| Helm | v3.10+ |
| Resources | Minimum 6 vCPUs, 12Gi RAM across nodes |
| Storage | Default StorageClass with ReadWriteOnce support |

---

## Architecture

```
                        ┌─────────────────────────────────┐
                        │         Load Balancer            │
                        └────────────┬────────────────────┘
                                     │
                        ┌────────────▼────────────────────┐
                        │     argocd-server (x2)           │
                        │     (API + UI)                   │
                        └────────────┬────────────────────┘
                                     │
               ┌─────────────────────┼─────────────────────┐
               │                     │                     │
   ┌───────────▼──────┐  ┌───────────▼──────┐  ┌──────────▼───────┐
   │ argocd-repo-     │  │ argocd-app-      │  │ argocd-          │
   │ server (x2)      │  │ controller (x2)  │  │ applicationset   │
   │ (Git/Helm sync)  │  │ (Reconciler)     │  │ controller (x1)  │
   └──────────────────┘  └──────────────────┘  └──────────────────┘
                                     │
                        ┌────────────▼────────────────────┐
                        │      Redis HA (Sentinel)         │
                        │      3 replicas                  │
                        └─────────────────────────────────┘
```

---

## Lab Steps

### Step 1 — Create the ArgoCD Namespace

```bash
kubectl create namespace argocd
```

---

### Step 2 — Add the ArgoCD Helm Repository

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

Verify the repo is available:

```bash
helm search repo argo/argo-cd
```

---

### Step 3 — Create the HA Values File

Create a file named `argocd-ha-values.yaml`:

```yaml
# argocd-ha-values.yaml

global:
  domain: argocd.example.com

configs:
  params:
    server.insecure: false
  cm:
    # Enable HA mode
    application.resourceTrackingMethod: annotation

# ArgoCD Server — 2 replicas
server:
  replicas: 2
  autoscaling:
    enabled: false
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 256Mi
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app.kubernetes.io/name: argocd-server
            topologyKey: kubernetes.io/hostname

# Application Controller — 2 replicas (sharding)
controller:
  replicas: 2
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi
  env:
    - name: ARGOCD_CONTROLLER_REPLICAS
      value: "2"

# Repo Server — 2 replicas
repoServer:
  replicas: 2
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app.kubernetes.io/name: argocd-repo-server
            topologyKey: kubernetes.io/hostname

# Redis HA — using Sentinel
redis-ha:
  enabled: true
  redis:
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
  haproxy:
    enabled: true
    replicas: 3

# ApplicationSet Controller — 1 replica (leader election built-in)
applicationSet:
  replicas: 1

# Notifications Controller
notifications:
  enabled: true
```

---

### Step 4 — Install ArgoCD in HA Mode

```bash
helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 6.7.3 \
  --values argocd-ha-values.yaml \
  --wait
```

---

### Step 5 — Verify the Deployment

Check all pods are running:

```bash
kubectl get pods -n argocd
```

Expected output:

```
NAME                                               READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                    1/1     Running   0          2m
argocd-application-controller-1                    1/1     Running   0          2m
argocd-applicationset-controller-xxxxxxxx-xxxxx    1/1     Running   0          2m
argocd-notifications-controller-xxxxxxxx-xxxxx     1/1     Running   0          2m
argocd-redis-ha-haproxy-xxxxxxxx-xxxxx             1/1     Running   0          2m
argocd-redis-ha-server-0                           2/2     Running   0          2m
argocd-redis-ha-server-1                           2/2     Running   0          2m
argocd-redis-ha-server-2                           2/2     Running   0          2m
argocd-repo-server-xxxxxxxx-xxxxx                  1/1     Running   0          2m
argocd-repo-server-xxxxxxxx-yyyyy                  1/1     Running   0          2m
argocd-server-xxxxxxxx-xxxxx                       1/1     Running   0          2m
argocd-server-xxxxxxxx-yyyyy                       1/1     Running   0          2m
```

Verify HA Redis Sentinel is working:

```bash
kubectl exec -n argocd -it argocd-redis-ha-server-0 -c redis -- redis-cli -p 26379 SENTINEL masters
```

---

### Step 6 — Retrieve the Admin Password

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

> **Note:** Change this password immediately after first login.

---

### Step 7 — Access the ArgoCD UI

**Option A — Port-forward (testing only):**

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open: [https://localhost:8080](https://localhost:8080)

**Option B — Expose via LoadBalancer:**

```bash
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "LoadBalancer"}}'
```

```bash
kubectl get svc argocd-server -n argocd
```

---

### Step 8 — Configure ArgoCD CLI

Install the CLI:

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd-linux-amd64
sudo mv argocd-linux-amd64 /usr/local/bin/argocd
```

Login:

```bash
argocd login localhost:8080 \
  --username admin \
  --password <YOUR_PASSWORD> \
  --insecure
```

---

### Step 9 — Test HA Failover

**Simulate argocd-server pod failure:**

```bash
# Delete one server pod
kubectl delete pod -n argocd -l app.kubernetes.io/name=argocd-server --field-selector=status.phase=Running | head -1

# Watch recovery
kubectl get pods -n argocd -w
```

**Simulate Redis failover:**

```bash
# Check current Redis master
kubectl exec -n argocd argocd-redis-ha-server-0 -c redis -- \
  redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster

# Delete the Redis master pod (Sentinel will elect a new one)
kubectl delete pod -n argocd argocd-redis-ha-server-0

# Re-check master after ~30 seconds
kubectl exec -n argocd argocd-redis-ha-server-1 -c redis -- \
  redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
```

---

### Step 10 — Deploy a Sample Application

Create a test Application:

```yaml
# test-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Apply it:

```bash
kubectl apply -f test-app.yaml
```

Watch sync status:

```bash
argocd app get guestbook
argocd app sync guestbook
```

---

## HA Configuration Reference

| Component | HA Strategy | Replicas |
|---|---|---|
| argocd-server | Active-Active | 2+ |
| argocd-repo-server | Active-Active | 2+ |
| argocd-application-controller | Sharding | 2+ |
| Redis | Sentinel (Active-Passive) | 3 |
| HAProxy | Load Balancer for Redis | 3 |
| ApplicationSet Controller | Leader Election | 1+ |

---

## Troubleshooting

### Pods not spreading across nodes

Add `topologySpreadConstraints` to your values:

```yaml
server:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: argocd-server
```

### Redis Sentinel not electing a master

```bash
# Check Sentinel logs
kubectl logs -n argocd argocd-redis-ha-server-0 -c sentinel

# Force a failover (use with caution)
kubectl exec -n argocd argocd-redis-ha-server-0 -c redis -- \
  redis-cli -p 26379 SENTINEL failover mymaster
```

### Application controller sharding issues

Verify shard assignment:

```bash
kubectl logs -n argocd argocd-application-controller-0 | grep shard
kubectl logs -n argocd argocd-application-controller-1 | grep shard
```

---

## Cleanup

```bash
helm uninstall argocd -n argocd
kubectl delete namespace argocd
```

---

## References

- [ArgoCD HA Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#high-availability)
- [ArgoCD Helm Chart](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd)
- [Redis HA Helm Chart](https://github.com/DandyDeveloper/charts/tree/master/charts/redis-ha)
