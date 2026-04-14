# ArgoCD — Deploy & Sync an Application
**Kind Cluster | kubectl | GitHub | GitOps**  
*End-to-End Setup & Application Deployment*  
*February 2026*

---

## 7.1 Create a New Application via the ArgoCD UI

**1.** Click **+ NEW APP** on the Applications page.

**2.** Fill in **General**:

| Field | Value |
|-------|-------|
| **Application Name** | `nginx-app` |
| **Project** | Your project name |
| **Sync Policy** | Automated |

**3.** Fill in **Source**:

| Field | Value |
|-------|-------|
| **Repository URL** | `https://github.com/<your-username>/nginx-argoproj.git` |
| **Revision** | `HEAD` 
| **Path** | `nginx-conf` |

**4.** Fill in **Destination**:

| Field | Value |
|-------|-------|
| **Cluster** | `https://kubernetes.default.svc` |
| **Namespace** | Leave it blank |

**5.** Click **Create**.

The application card will appear — it may show **Out of Sync** if auto-sync is off.

---

## 7.2 Sync the Application

Click **Sync** on the application card (or inside the app detail view).

ArgoCD will pull the manifests from GitHub, render them, and apply them to the cluster.

> ✅ Once sync completes, the application status will update to **Healthy** and **Synced**.
