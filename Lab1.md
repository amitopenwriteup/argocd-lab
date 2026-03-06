# ArgoCD Workshop Guide
**Kind Cluster | kubectl | GitHub | GitOps**  
*End-to-End Setup & Application Deployment*  
*February 2026*

---

## Table of Contents

1. [Prerequisites & Tools](#prerequisites--tools) — Page 2
2. [Kind Cluster Setup](#kind-cluster-setup) — Page 2
3. [Install ArgoCD](#install-argocd) — Page 3
4. [Access ArgoCD Server & Password](#access-argocd-server--retrieve-password) — Page 3

---

## 1. Prerequisites & Tools

Make sure the following tools are installed and available on your local machine before starting the workshop.

| Tool | Purpose | Install Reference |
|------|---------|-------------------|
| Docker | Container runtime for Kind | docker.com |
| Kind | Kubernetes in Docker | kind.sigs.k8s.io |
| kubectl | Kubernetes CLI | kubernetes.io/docs/tasks |
| curl | Download & test endpoints | Pre-installed (Linux/Mac) |
| Git | Clone / fork repositories | git-scm.com |
| Browser | Access ArgoCD UI | Chrome / Firefox / Edge |

> 💡 **Tip:** All commands in this guide are written for Linux / macOS terminals. Windows users should use WSL 2 or Git Bash.

---

## 2. Kind Cluster Setup

### Setup Docker

```bash
# Add Docker's official GPG key:
sudo su
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Kind (Kubernetes IN Docker) lets you run a full Kubernetes cluster inside Docker containers — perfect for local development and workshops.

### Install kubectl

```bash
curl -LO https://dl.k8s.io/release/v1.35.0/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version
```

### Install Kind

```bash
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.31.0/kind-linux-amd64

# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.31.0/kind-linux-arm64

chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

---

### 2.1 Create a Kind Cluster

Run the commands below to spin up a single-node cluster named `argocd-workshop`:

```bash
vi config.yaml
```

Copy and paste the following cluster configuration:

```yaml
# 4 node (3 workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.30.0
- role: worker
  image: kindest/node:v1.30.0
- role: worker
  image: kindest/node:v1.30.0
- role: worker
  image: kindest/node:v1.30.0
```

Save the file, then run:

```bash
kind create cluster --config config.yaml
```

---

### 2.2 Verify the Cluster

```bash
kubectl cluster-info
kubectl get nodes
```

---

### 2.3 Set the Correct Context

```bash
kubectl config get-contexts
```

---

## 3. Install ArgoCD on the Cluster

We install ArgoCD into its own dedicated namespace using the official manifest provided by the ArgoCD project.

**1. Create the ArgoCD namespace:**

```bash
kubectl create namespace argocd
```

**2. Apply the ArgoCD install manifest:**

```bash
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**3. Wait for all pods to be Running:**

```bash
kubectl -n argocd get pods
```

> ⏳ **Note:** It may take 2–3 minutes for all ArgoCD pods (`server`, `application-controller`, `repo-server`, `dex`, `redis`) to reach `Running` status. Do not proceed until all are Running.

---

## 4. Access ArgoCD Server & Retrieve Password

### 4.1 Port-Forward the ArgoCD Server

Since Kind uses a local cluster, we port-forward to expose the ArgoCD UI on your machine:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 \
  --address 0.0.0.0 > argocd-portforward.log 2>&1 &
```

Leave this terminal open. Open a **new terminal** for remaining commands.

---

### 4.2 Open the ArgoCD UI

Get your machine's IP address:

```bash
hostname -i
```

Open your browser and go to: `https://<your-ip>:8080`

---

### 4.3 Retrieve the Initial Admin Password

ArgoCD auto-generates a password stored in a Kubernetes secret. Decode it with:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```

> 🔐 **Login Credentials**  
> **Username:** `admin`  
> **Password:** `<output of the command above>`  
> Copy the decoded password before logging in.

---

### 4.4 Log In

Enter `admin` as the username and paste the decoded password.

**You are now inside the ArgoCD dashboard.** 🎉
