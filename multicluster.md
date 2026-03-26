# ArgoCD Remote KinD Cluster Registration — Step-by-Step

---

## Step 1 — Increase inotify Limits

```bash
sudo sysctl fs.inotify.max_user_instances=512
sudo sysctl fs.inotify.max_user_watches=524288
```

KinD nodes run systemd inside Docker and consume inotify instances heavily. The default kernel limit of 128 is exhausted by the first cluster, so we raise it before creating a second one.

---

## Step 2 — Create the Remote KinD Cluster

```bash
kind create cluster --name argocd-remote
```

This provisions a new single-node Kubernetes cluster named `argocd-remote` using Docker. It runs independently from your existing `argocd-control` cluster where ArgoCD is installed.

---

## Step 3 — Get the Docker Internal IP

```bash
docker inspect argocd-remote-control-plane \
  --format '{{ .NetworkSettings.Networks.kind.IPAddress }}'
```

ArgoCD pods run inside Docker and cannot reach `127.0.0.1` (the host). We need the container's internal Docker bridge IP (e.g., `172.18.0.4`) so ArgoCD can actually connect to the remote cluster's API server.

---

## Step 4 — Patch the Kubeconfig with the Docker IP

```bash
kubectl config view --raw --minify \
  --context kind-argocd-remote \
  | sed 's|https://127.0.0.1:[0-9]*|https://172.18.0.4:6443|g' \
  > /tmp/argocd-remote-kubeconfig.yaml
```

The default kubeconfig for KinD uses `127.0.0.1:<random-port>` as the API server address, which is only reachable from the host. This command exports a patched copy with the Docker IP substituted in so ArgoCD can reach it from within the cluster network.

---

## Step 5 — Verify the Patched Server Address

```bash
grep server /tmp/argocd-remote-kubeconfig.yaml
# Expected: server: https://172.18.0.4:6443
```

A quick sanity check to confirm the sed substitution worked correctly before handing the kubeconfig to ArgoCD. If it still shows `127.0.0.1`, re-check your Docker IP from Step 3.

---

## Step 6 — Register the Remote Cluster with ArgoCD

```bash
argocd cluster add kind-argocd-remote \
  --name argocd-remote \
  --server localhost:8080 \
  --insecure \
  --kubeconfig /tmp/argocd-remote-kubeconfig.yaml
```

This creates a `ServiceAccount`, `ClusterRole`, and `ClusterRoleBinding` in the remote cluster and stores its credentials in ArgoCD. The `--kubeconfig` flag ensures ArgoCD uses the patched address instead of the broken `127.0.0.1` from the default kubeconfig.

---

## Step 7 — Verify Both Clusters Are Registered

```bash
argocd cluster list
```

Confirms ArgoCD can reach both the local in-cluster target and the newly registered remote cluster. Both should show `Successful` status — if the remote shows `Unknown`, double-check the Docker IP in Step 3.

**Expected output:**

```
SERVER                           NAME           STATUS
https://kubernetes.default.svc   in-cluster     Successful
https://172.18.0.4:6443          argocd-remote  Successful
```

---

## Summary

| Step | What it does |
|------|-------------|
| 1 — inotify limits | Prevents kernel resource exhaustion when running 2+ KinD clusters |
| 2 — Create cluster | Spins up the second KinD cluster (`argocd-remote`) |
| 3 — Docker IP | Finds the container-internal IP reachable by ArgoCD pods |
| 4 — Patch kubeconfig | Replaces `127.0.0.1` with the Docker IP in a local kubeconfig copy |
| 5 — Verify kubeconfig | Confirms the patched address is correct before proceeding |
| 6 — Register cluster | Adds the remote cluster to ArgoCD using the patched kubeconfig |
| 7 — Verify | Confirms both clusters show `Successful` in ArgoCD |
