# ArgoCD Sync Policy — Lab 4: Server-Side Apply & Apply Out of Sync Only
**Kind Cluster | kubectl | GitHub | GitOps**  
*End-to-End Setup & Application Deployment*  
*February 2026*

---

## Objective

Test **Server-Side Apply** and **Apply Out of Sync Only** for large applications and selective syncing.

---

## Create the Application in ArgoCD UI

**1.** Click **+ NEW APP**

**2.** Fill in **General**:

| Field | Value |
|-------|-------|
| **Application Name** | `lab4-advanced` |
| **Project** | `default` |
| **Sync Policy** | Automatic |

**3.** Fill in **Source**:

| Field | Value |
|-------|-------|
| **Repository URL** | `<YOUR_REPO_URL>` |
| **Revision** | `HEAD` |
| **Path** | `lab4-advanced-options` |

**4.** Fill in **Destination**:

| Field | Value |
|-------|-------|
| **Cluster URL** | `https://kubernetes.default.svc` |
| **Namespace** | `lab4` |

**5.** Configure **Sync Policy**:

| Option | Setting |
|--------|---------|
| ✅ Enable Auto-Sync | Checked |
| ✅ Prune Resources | Checked |
| ⬜ Self Heal | Unchecked (for testing) |

**6.** Configure **Sync Options**:

| Option | Setting |
|--------|---------|
| ✅ Auto-Create Namespace | Checked |
| ⬜ Apply Out of Sync Only | Unchecked initially |
| ⬜ Server-Side Apply | Unchecked initially |

**7.** Click **Create** and **Sync**.

---

## Test Scenarios

### Test 1 — Apply Out of Sync Only

Verify all resources are running:

```bash
kubectl get all,cm -n lab4
```

Modify a single deployment:

```bash
cd lab4-advanced-options
sed -i 's/replicas: 2/replicas: 3/' deployment-1.yaml
git add deployment-1.yaml
git commit -m "Scale app-1 to 3 replicas"
git push
```

**Without Apply Out of Sync Only:**

- Click **Refresh** and **Sync**
- ArgoCD applies **all** resources
- Observe sync logs — every resource is processed

**With Apply Out of Sync Only:**

1. Go to **App Details → Edit**
2. Check ✅ **Apply Out of Sync Only** → **Save**

Make another change:

```bash
sed -i 's/replicas: 3/replicas: 4/' deployment-1.yaml
git add deployment-1.yaml
git commit -m "Scale app-1 to 4 replicas"
git push
```

- Click **Refresh** and **Sync**
- Only `deployment-1` is applied — unchanged resources are skipped
- Significantly faster for applications with 100+ resources

---

### Test 2 — Server-Side Apply

Create a ConfigMap with large annotations that can exceed client-side apply limits:

```bash
cat > large-configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: large-config
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ConfigMap","metadata":{"name":"large-config","namespace":"lab4"},"data":{"key":"value"}}
data:
  large-data: |
    [database]
    host=localhost
    port=5432
    database=myapp

    [cache]
    host=redis
    port=6379

    [logging]
    level=info
    format=json
EOF

git add large-configmap.yaml
git commit -m "Add large configmap"
git push
```

**Test client-side apply (default):**

- Sync the application — observe any warnings with large resources

**Test server-side apply:**

1. Go to **App Details → Edit**
2. Check ✅ **Server-Side Apply** → **Save**
3. Click **Sync**

**Observe:**

- No annotation size limitations
- Better conflict resolution
- Larger resources are handled without errors

---

### Test 3 — Combined Options Performance

Make changes to multiple resources:

```bash
sed -i 's/version: v1/version: v2/' deployment-1.yaml
sed -i 's/version: v1/version: v2/' deployment-2.yaml
sed -i 's/app.version=1.0.0/app.version=2.0.0/' configmap-1.yaml

git add .
git commit -m "Update multiple resources to v2"
git push
```

**Configuration A — Default (no options):**

- Uncheck all sync options → **Sync**
- Note all resources processed and total time

**Configuration B — With optimisations:**

- Check ✅ **Apply Out of Sync Only**
- Check ✅ **Server-Side Apply**
- **Sync** and compare — fewer resources processed, faster completion

---

### Test 4 — Conflict Resolution

Create a conflict scenario:

```bash
# Add a manual annotation directly on the cluster
kubectl annotate deployment app-1 -n lab4 manual-annotation=test

# Update the same deployment in Git
sed -i 's/replicas: 4/replicas: 5/' deployment-1.yaml
git add deployment-1.yaml
git commit -m "Scale to 5 replicas"
git push
```

**With Server-Side Apply:**

- Changes are merged instead of replaced
- Manual annotation is preserved alongside the Git-defined state

**Without Server-Side Apply:**

- Manual changes may be overwritten
- Full resource replacement

---

## Questions to Answer

1. When is **Apply Out of Sync Only** most beneficial?
2. What are the advantages of **Server-Side Apply**?
3. How do these options affect sync performance?
4. What is the downside of **Apply Out of Sync Only**?

---

## Cleanup

In the ArgoCD UI: click **lab4-advanced → Delete → Confirm**.
