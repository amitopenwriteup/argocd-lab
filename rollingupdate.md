# LAB 2: Rolling Update Strategies in ArgoCD

---

## SECTION 0: Participant Setup — Fork & Clone

**Step 1 — Fork the repository**

```
1. Open browser → https://github.com/amitopenwriteup/argocd-rolling
2. Click "Fork" button (top right)
3. Select your GitHub account as destination
4. Click "Create fork"
```

**Step 2 — Copy your SSH clone URL**

```
Your forked repo → Click "Code" button → SSH tab
→ Copy: git@github.com:<your-username>/argocd-rolling.git
```

**Step 3 — Ensure SSH key is set up**

```bash
# Check if SSH key exists
ls ~/.ssh/id_rsa.pub || ls ~/.ssh/id_ed25519.pub

# If not, generate one
ssh-keygen -t ed25519 -C "your-email@example.com"

# Copy public key and add to GitHub
cat ~/.ssh/id_ed25519.pub
# GitHub → Settings → SSH and GPG keys → New SSH key → Paste
```

**Step 4 — Clone your forked repo**

```bash
git clone git@github.com:<your-username>/argocd-rolling.git
cd argocd-rolling
```

**Step 5 — Verify repo contents**

```bash
ls -la
# Expected:
#   deployment.yaml   ← already provided
#   service.yaml      ← already provided
```

**Step 6 — Set git identity**

```bash
git config user.name "Your Name"
git config user.email "your-email@example.com"
```

---

## SECTION 1: Create Application from ArgoCD UI

**Step 1 — Open ArgoCD UI**

```
https://<your-argocd-server>
```

**Step 2 — Click `+ NEW APP`** (top left corner)

**Step 3 — Fill in the form**

**GENERAL**

| Field            | Value            |
|------------------|------------------|
| Application Name | `argocd-rolling` |
| Project Name     | `default`        |
| Sync Policy      | `Manual`         |

**SOURCE**

| Field          | Value                                               |
|----------------|-----------------------------------------------------|
| Repository URL | `https://github.com/<username>/argocd-rolling.git` |
| Revision       | `HEAD`                                              |
| Path           | `.`                                                 |

**DESTINATION**

| Field       | Value                            |
|-------------|----------------------------------|
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace   | `default`                        |

**Step 4 — Click `CREATE`**

**Step 5 — Verify app card appears**

```
Status: Unknown / OutOfSync  ← expected, not yet synced
```

---

## SECTION 2: Initial Sync — Deploy v1 Baseline

**Step 1 — Sync v1 from UI**

```
App Detail View → SYNC → SYNCHRONIZE
→ Watch 4 pods turn green in App Tree
```

**Step 2 — Verify from CLI**

```bash
kubectl rollout status deployment/rolling-app

kubectl get pods -l app=rolling-app
# Expected: 4 pods Running

kubectl get deployment rolling-app \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# Expected: nginx:1.19
```

---



## SECTION 5: Rollback Strategy 1 — ArgoCD UI Rollback

**Step 1 — Open History from UI**

```
App Detail View → HISTORY AND ROLLBACK (clock icon, top bar)

History table:
  ID  Revision   Message
  0   abc1234    v1: nginx:1.19 - baseline
  1   def5678    v2: nginx:1.20 - rolling update

```

