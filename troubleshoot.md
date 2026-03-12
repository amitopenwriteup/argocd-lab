# LAB 1: ArgoCD Troubleshooting

---

## SECTION 1: Create Application from ArgoCD UI

**Step 1 — Open ArgoCD UI**

```
https://<your-argocd-server>
```

**Step 2 — Click `+ NEW APP`** (top left corner)

**Step 3 — Fill in the form**

**GENERAL**

| Field            | Value                      |
|------------------|----------------------------|
| Application Name | `argocd-troubleshooting`   |
| Project Name     | `default`                  |
| Sync Policy      | `Manual`                   |

**SOURCE**

| Field          | Value                                                                 |
|----------------|-----------------------------------------------------------------------|
| Repository URL | `https://github.com/amitopenwriteup/argocd-troubleshooting.git`      |
| Revision       | `HEAD`                                                                |
| Path           | `.`                                                                   |

**DESTINATION**

| Field       | Value                               |
|-------------|-------------------------------------|
| Cluster URL | `https://kubernetes.default.svc`    |
| Namespace   | `default`                           |

**Step 4 — Click `CREATE`** (top left)

**Step 5 — Verify app card appears**

```
Status: Unknown / OutOfSync  ← expected, not yet synced
```

---

## SECTION 2: Initial Sync from UI

**Step 1 — Click on the app card** `argocd-troubleshooting`

**Step 2 — Click `SYNC` button** (top bar)

**Step 3 — Sync options popup**

| Option     | Value       |
|------------|-------------|
| Revision   | `HEAD`      |
| PRUNE      | `unchecked` |
| DRY RUN    | `unchecked` |
| APPLY ONLY | `unchecked` |
| FORCE      | `unchecked` |

**Step 4 — Click `SYNCHRONIZE`**

**Step 5 — Watch the sync**

```
App Tree view shows resources turning:
  Yellow  → Progressing
  Green   → Healthy / Synced
  Red     → Degraded / Failed   ← we will debug these
```

---

## SECTION 3: Out of Sync Debugging

**Step 1 — Identify OutOfSync resources from UI**

```
App Detail View → Click any Orange/Yellow resource
→ Shows DIFF tab with Live vs Desired state
```

**Step 2 — CLI verification**

```bash
# Overall app status
argocd app get argocd-troubleshooting

# See exact diff between git and cluster
argocd app diff argocd-troubleshooting

# Force refresh from git (bypass cache)
argocd app get argocd-troubleshooting --hard-refresh

# List orphaned resources
argocd app resources argocd-troubleshooting --orphaned
```

**Step 3 — Hard Refresh from UI**

```
App Detail View → Click ⟳ Refresh dropdown → Select "Hard Refresh"
```

---

## SECTION 4: Sync Failure Debugging

**Step 1 — Identify failed sync from UI**

```
App Detail View → SYNC STATUS shows "SyncFailed"
→ Click "More" next to sync status → shows error message
```


```

**Step 2 — Retry from UI**

```
App Detail View → SYNC → check "AUTO-CREATE NAMESPACE" → SYNCHRONIZE
```

---

## SECTION 5: Hook Failure Debugging

**Step 1 — Identify hook failure from UI**

```
App Detail View → Look for Job resource with  red icon
→ Click the Job → Click "LOGS" tab → Read error output
```

**Step 2 — CLI investigation**

```bash
# List hooks and their status
argocd app get argocd-troubleshooting -o json \
  | jq '.status.resources[] | select(.hook==true)'

# Get hook job logs
kubectl get jobs -n default


# Describe the failed job
kubectl describe job presync-hook

# Manually delete failed hook to unblock sync
kubectl delete job presync-hook
```

**Step 3 — Delete hook from UI to unblock**

```
App Detail View → Click failed Job → Click "DELETE" → Confirm
→ Then click SYNC again
```

---

## SECTION 6: Health Status Debugging

**Step 1 — Identify degraded resources from UI**

```
App Detail View → Red resources indicate Degraded health
→ Click resource → SUMMARY tab shows current state
→ Click EVENTS tab → shows K8s events (ImagePullBackOff etc)
```

