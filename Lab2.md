# ArgoCD Projects
**Kind Cluster | kubectl | GitHub | GitOps**  
*End-to-End Setup & Application Deployment*  
*February 2026*

---

## Create an ArgoCD Project

Projects in ArgoCD allow you to group applications, enforce access controls, and restrict source/destination clusters and namespaces.

### Via the ArgoCD UI

**1.** Go to **Settings → Projects**

**2.** Click **Create Project**

**3.** Fill in the project details:

| Field | Value |
|-------|-------|
| **Description** | Workshop project for nginx deployment |
| **Source Repositories** | `*` |
| **Destinations (Cluster)** | `*` |
| **Destinations (Namespace)** | `*` |

**4.** Click **Create**. The project is now active and visible in the **Projects** list.

---

### Cluster Allow List

> ℹ️ Setting Source Repositories and Destinations to `*` (wildcard) permits any repository and any cluster/namespace. In production environments, restrict these to specific repos, clusters, and namespaces to enforce least-privilege access.
