# NFS Setup on KIND Kubernetes
**Kind Cluster | kubectl | GitHub | GitOps**  
*End-to-End Setup & Application Deployment*  
*February 2026*

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Step 1 — Install NFS Server on Host Machine](#step-1--install-nfs-server-on-host-machine)
3. [Step 2 — Create NFS Share Directory](#step-2--create-nfs-share-directory)
4. [Step 3 — Configure NFS Exports](#step-3--configure-nfs-exports)
5. [Step 4 — Apply NFS Configuration](#step-4--apply-nfs-configuration)
6. [Step 5 — Get Host IP Address](#step-5--get-host-ip-address)
7. [Step 6 — Install NFS Client on KIND Nodes](#step-6--install-nfs-client-on-kind-nodes)
8. [Step 7 — Deploy NFS Provisioner in Kubernetes](#step-7--deploy-nfs-provisioner-in-kubernetes)
9. [Step 8 — Create StorageClass](#step-8--create-storageclass)
10. [Step 9 — Verify Setup](#step-9--verify-setup)
11. [Step 10 — Test NFS with PVC](#step-10--test-nfs-with-pvc)

---

## Prerequisites

- KIND cluster is running
- Root or sudo access to the host machine

---

## Step 1 — Install NFS Server on Host Machine

```bash
# Update package list
sudo apt-get update

# Install NFS server packages
sudo apt-get install -y nfs-kernel-server nfs-common

# Verify NFS server is installed
systemctl status nfs-server
```

---

## Step 2 — Create NFS Share Directory

```bash
# Create directory for NFS share
sudo mkdir -p /srv/nfs/kubedata

# Set permissions
sudo chown -R nobody:nogroup /srv/nfs/kubedata
sudo chmod 777 /srv/nfs/kubedata
```

---

## Step 3 — Configure NFS Exports

```bash
# Edit exports file
sudo nano /etc/exports
```

Add the following line (replace with your network range if needed):

```
/srv/nfs/kubedata 172.18.0.0/16(rw,sync,no_subtree_check,no_root_squash,insecure)
```

---

## Step 4 — Apply NFS Configuration

```bash
# Export the shared directory
sudo exportfs -a

# Restart NFS server
sudo systemctl restart nfs-kernel-server

# Verify exports
sudo exportfs -v
```

---

## Step 5 — Get Host IP Address

Get the host IP that KIND containers can reach:

```bash
ip addr show docker0 | grep inet
```

> ℹ️ The IP is typically `172.17.0.1` or similar. Note this value — it will be used in Step 7 when configuring the NFS provisioner.

---

## Step 6 — Install NFS Client on KIND Nodes

Install the NFS client on each KIND node:

```bash
for node in kind-control-plane kind-worker kind-worker2 kind-worker3; do
  docker exec -it $node bash -c \
    "export DEBIAN_FRONTEND=noninteractive && apt-get update && apt-get install -y nfs-server"
done
```

---

## Step 7 — Deploy NFS Provisioner in Kubernetes

```bash
# Create namespace
kubectl create namespace nfs-provisioner

# Install Helm if not already installed
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add the NFS provisioner Helm repository
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

# Install the provisioner (replace 172.18.0.1 with your host IP from Step 5)
helm install nfs-subdir-external-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace nfs-provisioner \
  --set nfs.server=172.18.0.1 \
  --set nfs.path=/srv/nfs/kubedata
```

---

## Step 8 — Create StorageClass

```bash
cat <<EOF > storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: cluster.local/nfs-subdir-external-provisioner
parameters:
  archiveOnDelete: "false"
EOF

kubectl apply -f storageclass.yaml
```

---

## Step 9 — Verify Setup

```bash
# Check if provisioner is running
kubectl get pods -n nfs-provisioner

# Check StorageClass
kubectl get storageclass

# Optional: Set as default StorageClass
kubectl patch storageclass nfs-client \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## Step 10 — Test NFS with PVC

### Create a test PersistentVolumeClaim

```bash
cat <<EOF > test-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f test-pvc.yaml

# Check PVC status
kubectl get pvc test-pvc
```

### Create a test Pod

```bash
cat <<EOF > test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: test
    image: nginx
    volumeMounts:
    - name: nfs-storage
      mountPath: /data
  volumes:
  - name: nfs-storage
    persistentVolumeClaim:
      claimName: test-pvc
EOF

kubectl apply -f test-pod.yaml

# Verify pod is running
kubectl get pod test-pod

# Check mounted volume
kubectl exec test-pod -- df -h /data
```

> ✅ Your NFS storage is now ready to use with your KIND Kubernetes cluster!
