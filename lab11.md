# ArgoCD Sync Policy — Lab 2: Prune Resources & Deletion Finalizer
**Kind Cluster | kubectl | GitHub | GitOps**  
*End-to-End Setup & Application Deployment*  
*February 2026*

---

## Objective

Test **Prune Resources** and **Set Deletion Finalizer** to understand resource cleanup behavior when manifests are removed from Git.

---

## Prerequisites

- ArgoCD installed and accessible
- Forked repo from Lab 1: `https://github.com/amitopenwriteup/argoappsynclab.git`

---

## Create the Application in ArgoCD UI

**1.** Click **+ NEW APP**

**2.** Fill in **General**:

| Field | Value |
|-------|-------|
| **Application Name** | `lab2-prune` |
| **Project** | `default` |
| **Sync Policy** | Automatic |

**3.** Fill in **Source**:

| Field | Value |
|-------|-------|
| **Repository URL** | `<YOUR_REPO_URL>` |
| **Revision** | `HEAD` |
| **Path** | `lab1-autosync` |

**4.** Fill in **Destination**:

| Field | Value |
|-------|-------|
| **Cluster URL** | `https://kubernetes.default.svc` |
| **Namespace** | *(leave blank)* |

**5.** Configure **Sync Policy**:

| Option | Setting |
|--------|---------|
| ✅ Enable Auto-Sync | Checked |
| ✅ Prune Resources | Checked |
| ✅ Self Heal | Checked |
| ✅ Set Deletion Finalizer | Checked |

**6.** Configure **Sync Options**:

| Option | Setting |
|--------|---------|
| ✅ Auto-Create Namespace | Checked |

**7.** Click **Create**, then click **Sync** to deploy.

---

## Test Scenarios

### Test 1 — Verify All Resources Created

```bash
kubectl get all -n stage-nginx-namespace
```

---

### Test 2 — Test Prune Resources

First, configure SSH authentication for your GitHub account:

```bash
sudo su
ssh-keygen -t ed25519
# Press Enter for all prompts (no passphrase required)

# Copy your public key
cat /root/.ssh/id_ed25519.pub
```

Add the public key to GitHub under **Settings → SSH and GPG Keys → New SSH Key**.

Copy the SSH clone URL from your forked repo, then clone it:

```bash
git clone <your-ssh-url>
cd argoappsynclab/lab2-prune/templates
```

Remove the ConfigMap from Git:

```bash
rm configmap.yaml
git add .
git commit -m "remove cm"
git push origin main
```

**Observe in the ArgoCD UI:**

- ArgoCD detects the Git change
- Auto-sync triggers
- `ConfigMap` is automatically **pruned** (deleted) from the cluster
- Refresh the page if needed
- Application status remains **Synced**

---

### Test 3 — Add Resource Back

Restore the deleted ConfigMap using git revert:

```bash
git revert HEAD
# Press Ctrl+X then Enter to confirm

git push origin main
```

ArgoCD will detect the revert and re-create the ConfigMap on the next sync.

---

### Test 4 — Test Deletion Finalizer

In the ArgoCD UI:

**1.** Click on **lab2-prune → DELETE**

**2.** Select the **Foreground** deletion option

**Observe in the ArgoCD UI:**

- The deletion finalizer ensures Kubernetes resources are cleaned up in the correct order before the application is removed
- You will see the cleanup progress before the application is fully deleted

>  **What is a Deletion Finalizer?**  
> A finalizer is a pre-delete hook that prevents the application object from being removed until all associated cluster resources have been successfully cleaned up. Without it, deleting an ArgoCD application may leave orphaned resources on the cluster.
