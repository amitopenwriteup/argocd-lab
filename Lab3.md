# ArgoCD Workshop Guide
**Kind Cluster | kubectl | GitHub | GitOps**  
*End-to-End Setup & Application Deployment*  
*February 2026*

---

## Table of Contents

1. [GitHub Setup — Fork & Personal Access Token](#1-github-setup--fork--personal-access-token)
   - [1.1 Fork the Repository](#11-fork-the-repository)
   - [1.2 Generate a Personal Access Token (PAT)](#12-generate-a-personal-access-token-pat)
2. [Register GitHub Repo in ArgoCD](#2-register-github-repo-in-argocd)
3. [Create an ArgoCD Project](#3-create-an-argocd-project)
   - [6.1 Via the ArgoCD UI](#61-via-the-argocd-ui)
   - [6.2 Via CLI (Alternative)](#62-via-cli-alternative)

---

## 1. GitHub Setup — Fork & Personal Access Token

### 1.1 Fork the Repository

Go to the source repository on GitHub:
**[https://github.com/amitopenwriteup/nginx-argoproj](https://github.com/amitopenwriteup/nginx-argoproj)**

1. Click the **Fork** button (top-right corner of the page).
2. Select your own GitHub account as the owner.
3. Click **Create fork**.

You now have your own copy at:
`https://github.com/<your-username>/nginx-argoproj`

---

### 1.2 Generate a Personal Access Token (PAT)

ArgoCD will connect to your forked GitHub repo over HTTPS using a PAT for authentication.

1. Go to **GitHub → Settings → Developer Settings → Personal Access Tokens → Tokens (classic)**
2. Click **Generate new token (classic)**.
3. Enter a descriptive name, e.g. `argocd-workshop-pat`
4. Set expiration to **30 days** (or as needed).
5. Select scopes — check **repo** (full repository access).
6. Click **Generate token** and copy the token immediately.

> ⚠️ **Important:** You will not be able to see the token again after leaving this page. Copy it now and store it somewhere safe.

---

## 2. Register GitHub Repo in ArgoCD

In the ArgoCD UI, navigate to **Settings → Repositories → Connect**.

| Field | Value |
|-------|-------|
| **Type** | HTTPS |
| **Repository URL** | `https://github.com/<your-username>/nginx-argoproj.git` |
| **Username** | Your GitHub username |
| **Password** | The PAT you just generated |

Click **Connect**. A green success indicator confirms the repo is linked.

---

## 3. Create an ArgoCD Project

Projects in ArgoCD allow you to group applications, enforce access controls, and restrict source/destination clusters and namespaces.

### 6.1 Via the ArgoCD UI

1. Go to **Settings → Projects**
2. Click **Create Project**
3. Fill in the project details:

| Field | Value |
|-------|-------|
| **Name** | `nginx-project` |
| **Description** | Workshop project for nginx deployment |
| **Source Repositories** | `*` |
| **Destinations (Cluster)** | `*` |
| **Destinations (Namespace)** | `*` |

4. Click **Create**. The project is now active and visible in the **Projects** list.

> ℹ️ **Cluster Allow List**  
> Setting Source Repositories and Destinations to `*` (wildcard) permits any repository and any cluster/namespace. In production environments, restrict these to specific repos, clusters, and namespaces to enforce least-privilege access.

---

### 6.2 Via CLI (Alternative)

If you prefer the command line, you can create the same project using:

```bash
argocd proj list
```

```bash
argocd proj create nginx-cli-project \
  --description "Workshop project for nginx deployment" \
  --src "https://github.com/amitopenwriteup/nginx-argoproj.git" \
  --dest-serviceaccounts false \
  --dest https://kubernetes.default.svc,default
```

Verify the project was created:

```bash
argocd proj list
argocd proj get nginx-cli-project
```
