# ArgoCD Sync Policy — Lab 1: Auto-Sync & Self-Heal
**Kind Cluster | kubectl | GitHub | GitOps**  
*End-to-End Setup & Application Deployment*  
*February 2026*

---

## Objective

Test **Enable Auto-Sync** and **Self-Heal** options to understand automated synchronization and drift correction.

---

## Prerequisites

- ArgoCD installed and accessible
- GitHub repository with sample application

---

## Setup

### Fork the Repository

Fork the following repo to your own GitHub account:

**[https://github.com/amitopenwriteup/argoappsynclab.git](https://github.com/amitopenwriteup/argoappsynclab.git)**

Map the forked repo as a source in your ArgoCD project.

---

## Create the Application in ArgoCD UI

**1.** Click **+ NEW APP**

**2.** Fill in **General**:

| Field | Value |
|-------|-------|
| **Application Name** | `lab1-autosync` |
| **Project** | `myproj` (your project name) |
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
| ⬜ Prune Resources | Unchecked |
| ✅ Self Heal | Checked |

**6.** Configure **Sync Options**:

| Option | Setting |
|--------|---------|
| ✅ Auto-Create Namespace | Checked |

**7.** Click **Create**.

---

## Test Scenarios

### Test 1 — Auto-Sync on Git Changes

In your forked GitHub repo, navigate to the path `lab1-autosync` and edit `deployment.yaml`:

```
replicas: 2  →  replicas: 3
```

**Observe in the ArgoCD UI:**

- Application shows **Syncing** status with a progress bar
- ArgoCD auto-syncs within ~3 minutes (default polling interval)
- Status updates to **Synced** and **Healthy** once complete

> ℹ️ If the sync does not trigger immediately, press **Refresh** in the UI.

---

### Test 2 — Self-Heal: Manual Cluster Change

Manually scale the deployment to drift from the Git-defined state:

```bash
kubectl scale deployment nginx -n stage-nginx-namespace --replicas=5

# Watch the deployment
kubectl get deployment nginx -n stage-nginx-namespace -w
```

**Observe in the ArgoCD UI:**

- Application briefly shows **OutOfSync**
- Self-heal triggers automatically
- Deployment reverts to **3 replicas** (as defined in Git)
- Status returns to **Synced**

---

## Cleanup

In the ArgoCD UI:

Click **lab1-autosync → DELETE → Confirm**
