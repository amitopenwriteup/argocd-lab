# Lab: ArgoCD SSO with GitHub via Dex

## Overview

Integrate ArgoCD with GitHub OAuth using Dex as the identity broker. After completing this lab, users will log in to ArgoCD using their GitHub account.

---

## Prerequisites

- ArgoCD installed and running (any namespace)
- `kubectl` access to the cluster
- A GitHub account with admin access to an Organization (or personal OAuth app)
- ArgoCD accessible via a public/reachable URL (e.g., `https://argocd.example.com`)

---

## Part 1: GitHub Side — Create an OAuth App

### Step 1: Go to GitHub OAuth Apps

- **For a GitHub Organization:**
  `https://github.com/organizations/<YOUR_ORG>/settings/applications`

- **For a personal account:**
  `https://github.com/settings/developers` → **OAuth Apps**

---

### Step 2: Register a New OAuth App

Click **"New OAuth App"** and fill in:

| Field | Value |
|---|---|
| **Application name** | `ArgoCD` (or any name) |
| **Homepage URL** | `https://argocd.example.com` |
| **Authorization callback URL** | `https://argocd.example.com/api/dex/callback` |

> ⚠️ The callback URL **must** end with `/api/dex/callback` — this is what Dex listens on.

Click **"Register application"**.

---

### Step 3: Copy the Credentials

After registration, GitHub shows you:

- ✅ **Client ID** → copy this (e.g., `Ov23liABC1234567890`)
- Click **"Generate a new client secret"**
- ✅ **Client Secret** → copy immediately (shown only once!)

---

## Part 2: Kubernetes Side — Store the Secret

Store the GitHub client secret in the `argocd-secret` Kubernetes Secret:

```bash
# Base64-encode the client secret
echo -n "YOUR_GITHUB_CLIENT_SECRET" | base64

# Patch the argocd-secret
kubectl patch secret argocd-secret \
  -n argocd \
  --type merge \
  -p '{"data": {"dex.github.clientSecret": "<BASE64_ENCODED_SECRET>"}}'
```

Verify:

```bash
kubectl get secret argocd-secret -n argocd -o jsonpath='{.data.dex\.github\.clientSecret}' | base64 -d
```

---

## Part 3: ArgoCD Side — Configure Dex

Edit the `argocd-cm` ConfigMap:

```bash
kubectl edit configmap argocd-cm -n argocd
```

Add the following under `data:`:

```yaml
data:
  url: https://argocd.example.com   # Must match your ArgoCD URL

  dex.config: |
    connectors:
    - type: github
      id: github
      name: GitHub
      config:
        clientID: Ov23liABC1234567890        # From GitHub OAuth App
        clientSecret: $dex.github.clientSecret  # References the K8s secret
        redirectURI: https://argocd.example.com/api/dex/callback
        orgs:
        - name: your-github-org              # Restrict to your GitHub Org
```

> 💡 Remove the `orgs` block if you want to allow **any** GitHub user (not recommended for production).

---

## Part 4: Configure RBAC

Edit the `argocd-rbac-cm` ConfigMap to map GitHub teams/users to ArgoCD roles:

```bash
kubectl edit configmap argocd-rbac-cm -n argocd
```

```yaml
data:
  policy.default: role:readonly    # Default role for all authenticated users

  policy.csv: |
    # Map a GitHub Org team to ArgoCD admin role
    g, your-github-org:platform-team, role:admin

    # Map a specific GitHub user to admin
    g, octocat, role:admin

    # Map another team to readonly
    g, your-github-org:dev-team, role:readonly
```

---

## Part 5: Restart ArgoCD Components

Apply changes by restarting the relevant pods:

```bash
kubectl rollout restart deployment argocd-server -n argocd
kubectl rollout restart deployment argocd-dex-server -n argocd
```

Wait for pods to be ready:

```bash
kubectl get pods -n argocd -w
```

---

## Part 6: Test the Login

1. Open `https://argocd.example.com` in your browser
2. Click **"LOG IN VIA GITHUB"**
3. GitHub will ask you to authorize the OAuth App → click **"Authorize"**
4. You are redirected back to ArgoCD and logged in 🎉

---

## Troubleshooting

| Issue | Fix |
|---|---|
| `invalid_client` error on GitHub | Check Client ID and Secret are correct |
| Redirect URI mismatch | Ensure callback URL exactly matches GitHub OAuth App setting |
| User gets `role:readonly` unexpectedly | Check `policy.csv` — GitHub username/team must match exactly |
| Dex pod crashing | Check logs: `kubectl logs -n argocd deploy/argocd-dex-server` |
| Login loop / 502 | Ensure `url:` in `argocd-cm` matches the actual ArgoCD URL |

---

## What You Need from GitHub — Summary Checklist

```
[ ] GitHub OAuth App created
[ ] Homepage URL set to ArgoCD URL
[ ] Callback URL set to: https://<argocd-url>/api/dex/callback
[ ] Client ID copied
[ ] Client Secret generated and copied (shown only once!)
[ ] (Optional) GitHub Organization name noted for team-based RBAC
[ ] (Optional) GitHub Team names noted for RBAC mapping
```

---

## Architecture Flow

```
User Browser
    │
    ▼
ArgoCD UI (https://argocd.example.com)
    │
    ▼
Dex (argocd-dex-server)
    │  ── redirects to ──▶  GitHub OAuth
    │                            │
    │  ◀── returns token ───────┘
    │
    ▼
ArgoCD RBAC check (argocd-rbac-cm)
    │
    ▼
Access Granted ✅
```
