# ArgoCD Sync Policy Labs 2

---

## Lab 3: Sync Options Testing

### Objective

Test **SKIP SCHEMA VALIDATION**, **PRUNE LAST**, **RESPECT IGNORE DIFFERENCES**, and **REPLACE** options.

---

### Setup Steps

#### 1. Create Application Repository

#### 2. Create Application in ArgoCD UI — Configuration 1

First, test without options:

1. Click **"+ NEW APP"**

**GENERAL:**
- Application Name: `lab3-sync-options`
- Project: `myproj`
- Sync Policy: `Manual` (we'll sync manually to see effects)

**SOURCE:**
- Repository URL: `<YOUR_REPO_URL>`
- Revision: `HEAD`
- Path: `lab3-sync-options`

**DESTINATION:**
- Cluster URL: `https://kubernetes.default.svc`
- Namespace: `lab3`

**SYNC POLICY:**
- ⬜ All options unchecked initially

**SYNC OPTIONS:**
- ⬜ SKIP SCHEMA VALIDATION (unchecked)
- ⬜ PRUNE LAST (unchecked)
- ⬜ RESPECT IGNORE DIFFERENCES (unchecked)
- ✅ AUTO-CREATE NAMESPACE
- ⬜ APPLY OUT OF SYNC ONLY
- ⬜ SERVER-SIDE APPLY
- ⬜ REPLACE

2. Click **CREATE**
3. Click **SYNC** → **SYNCHRONIZE**

---

### Test Scenarios

#### Test 1: SKIP SCHEMA VALIDATION

```bash
cd lab3-sync-options/templates

# Create a resource with intentional schema issue (wrong field name)
cat > bad-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bad
  template:
    metadata:
      labels:
        app: bad
    spec:
      containers:
      - name: app
        image: nginx:1.21
        wrongField: "this will fail validation"
EOF

git add bad-deployment.yaml
git commit -m "Add deployment with validation error"
git push
```

**Observe in UI:**
- Click **REFRESH** in ArgoCD UI
- Try to **SYNC** — you'll see a validation error
- Go to app settings: **APP DETAILS** → **EDIT** → **SYNC OPTIONS**
- ✅ Check **SKIP SCHEMA VALIDATION** → Click **SAVE**
- Try **SYNC** again — validation will be skipped (though it may still fail at apply)

---

#### Test 2: PRUNE LAST

```bash
# Remove the bad deployment and create proper multi-resource setup
git rm bad-deployment.yaml
git commit -m "Remove bad deployment"
git push

# Create configmap
cat > configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-config
data:
  app.properties: |
    app.name=demo-app
    app.env=dev
    app.version=1.0.0
  welcome.txt: |
    Welcome to the demo application!
    This file is mounted from ConfigMap.
EOF

git add configmap.yaml
git commit -m "Add cm"
git push

# Sync in ArgoCD UI

# Add a resource that depends on configmap
cat > deployment-with-dependency.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-config
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-user
  template:
    metadata:
      labels:
        app: config-user
    spec:
      containers:
      - name: app
        image: nginx:1.21
        volumeMounts:
        - name: config
          mountPath: /etc/config
      volumes:
      - name: config
        configMap:
          name: demo-config
EOF

git add deployment-with-dependency.yaml
git commit -m "Add deployment with configmap dependency"
git push

# Now remove the configmap
git rm configmap.yaml
git commit -m "Remove configmap (will cause issues)"
git push
```

**Observe in UI:**
- Enable **PRUNE RESOURCES** in sync policy
- Enable **PRUNE LAST** in sync options
- Click **SYNC**
- With PRUNE LAST, dependent resources are deleted before the configmap — this prevents errors from missing dependencies

---

#### Test 3: RESPECT IGNORE DIFFERENCES

```bash
# In ArgoCD UI:
# 1. Click app → APP DETAILS → EDIT
# 2. Scroll to bottom, add IGNORE DIFFERENCES section (use YAML mode)
```

Add this configuration in YAML mode:

```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
      - /spec/replicas
```

```bash
# Manually scale the deployment
kubectl scale deployment demo-app -n lab3 --replicas=5

# Check UI — app should show OutOfSync
# Now enable RESPECT IGNORE DIFFERENCES in sync options
# Click SYNC
```

**Observe in UI:**
- With **RESPECT IGNORE DIFFERENCES** enabled, replicas difference is ignored
- App shows as **"Synced"** despite difference
- Without it, ArgoCD would sync back to 2 replicas

---

#### Test 4: REPLACE Option

```bash
# Create a resource that will need replacement
# Create a headless service for the StatefulSet
cat > service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-stateless
spec:
  selector:
    app: nginx-stateful
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  type: ClusterIP
EOF

# Create a resource that will need replacement
cat > statefulset.yaml << 'EOF'
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-stateful
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx-stateful
  template:
    metadata:
      labels:
        app: nginx-stateful
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
          name: web
EOF

git add service.yaml statefulset.yaml
git commit -m "Add statefulset and stateless service"
git push

# Sync first, then modify immutable field
sed -i 's/serviceName: "nginx"/serviceName: "nginx-service"/' statefulset.yaml
git add statefulset.yaml
git commit -m "Modify immutable field"
git push

# Sync first, then modify immutable field
sed -i 's/serviceName: "nginx"/serviceName: "nginx-service"/' statefulset.yaml
git add statefulset.yaml
git commit -m "Modify immutable field"
git push
```

**Observe in UI:**
- Try to **SYNC** without REPLACE — you'll get an error (immutable field)
- Enable **REPLACE** option
- Click **SYNC** — resource will be deleted and recreated

---

### Cleanup

Delete application from UI.

---

## Lab 4: Advanced Options — Server-Side Apply and Apply Out of Sync Only

### Objective

Test **SERVER-SIDE APPLY** and **APPLY OUT OF SYNC ONLY** options for large applications and selective syncing.

---

### Setup Steps

#### 1. Create Large Application Repository

#### 2. Create Application in ArgoCD UI

1. Click **"+ NEW APP"**

**GENERAL:**
- Application Name: `lab4-advanced`
- Project: `default`
- Sync Policy: `Automatic`

**SOURCE:**
- Repository URL: `<YOUR_REPO_URL>`
- Revision: `HEAD`
- Path: `lab4-advanced-options`

**DESTINATION:**
- Cluster URL: `https://kubernetes.default.svc`
- Namespace: `lab4`

**SYNC POLICY:**
- ✅ ENABLE AUTO-SYNC
- ✅ PRUNE RESOURCES
- ⬜ SELF HEAL (unchecked for testing)

**SYNC OPTIONS:**
- ✅ AUTO-CREATE NAMESPACE
- ⬜ APPLY OUT OF SYNC ONLY (unchecked initially)
- ⬜ SERVER-SIDE APPLY (unchecked initially)

2. Click **CREATE** and **SYNC**

---

### Test Scenarios

#### Test 1: APPLY OUT OF SYNC ONLY

```bash
# Verify all resources are deployed
kubectl get all,cm -n lab4

# Modify only ONE deployment
cd lab4-advanced-options
sed -i 's/replicas: 2/replicas: 3/' deployment-1.yaml
git add deployment-1.yaml
git commit -m "Scale app-1 to 3 replicas"
git push
```

**Without APPLY OUT OF SYNC ONLY:**
- Click **REFRESH** and **SYNC**
- ArgoCD applies ALL resources
- Observe in sync logs: all resources are processed

**With APPLY OUT OF SYNC ONLY:**
- Go to **APP DETAILS** → **EDIT**
- ✅ Check **APPLY OUT OF SYNC ONLY** → Click **SAVE**
- Make another change:

```bash
sed -i 's/replicas: 3/replicas: 4/' deployment-1.yaml
git add deployment-1.yaml
git commit -m "Scale app-1 to 4 replicas"
git push
```

- Click **REFRESH** and **SYNC**
- Observe: only `deployment-1` is applied — much faster for large applications

---

#### Test 2: SERVER-SIDE APPLY

```bash
# Create a resource with large annotations
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
    # This is a large configuration file
    # that exceeds annotation size limits
    # when using client-side apply
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
    # ... imagine this continues for many lines ...
EOF

git add large-configmap.yaml
git commit -m "Add large configmap"
git push
```

**Test Client-Side Apply (default):**
- **SYNC** the application
- May see warnings or issues with large resources

**Test Server-Side Apply:**
- Go to **APP DETAILS** → **EDIT**
- ✅ Check **SERVER-SIDE APPLY** → Click **SAVE**
- Click **SYNC**

**Observe:**
- Server-side apply handles large resources better
- No annotation size limitations
- Better conflict resolution

---

#### Test 3: Combined Options Performance

```bash
# Make changes to multiple resources
sed -i 's/version: v1/version: v2/' deployment-1.yaml
sed -i 's/version: v1/version: v2/' deployment-2.yaml
sed -i 's/app.version=1.0.0/app.version=2.0.0/' configmap-1.yaml
git add .
git commit -m "Update multiple resources to v2"
git push
```

**Configuration A: Default (no options)**
- Uncheck all sync options → Click **SYNC**
- Measure time to complete and note resources processed

**Configuration B: With optimizations**
- ✅ APPLY OUT OF SYNC ONLY
- ✅ SERVER-SIDE APPLY
- Click **SYNC**
- Observe: fewer resources processed, faster completion

---

#### Test 4: Conflict Resolution

```bash
# Create a conflict scenario
kubectl annotate deployment app-1 -n lab4 manual-annotation=test

# Update the same deployment in Git
sed -i 's/replicas: 4/replicas: 5/' deployment-1.yaml
git add deployment-1.yaml
git commit -m "Scale to 5 replicas"
git push
```

**With Server-Side Apply:**
- Handles conflicts more gracefully
- Merges changes instead of replacing
- Manual annotation is preserved

**Without Server-Side Apply:**
- May overwrite manual changes
- Full resource replacement

---

### Questions to Answer

1. When is APPLY OUT OF SYNC ONLY most beneficial?
2. What are the advantages of SERVER-SIDE APPLY?
3. How do these options affect sync performance?
4. What's the downside of APPLY OUT OF SYNC ONLY?

---

### Cleanup

Delete application from UI.

---

## Summary and Best Practices

### Option Recommendations

| Option | When to Use | When NOT to Use |
|---|---|---|
| **ENABLE AUTO-SYNC** | Production apps with CI/CD | Apps needing manual approval |
| **PRUNE RESOURCES** | Clean environments | Shared namespaces |
| **SELF HEAL** | Prevent drift | Debugging / troubleshooting |
| **SET DELETION FINALIZER** | Ensure cleanup | Quick testing |
| **SKIP SCHEMA VALIDATION** | Custom CRDs, testing | Production (security risk) |
| **PRUNE LAST** | Dependent resources | Simple apps |
| **RESPECT IGNORE DIFFERENCES** | With `ignoreDifferences` configured | All resources should sync |
| **AUTO-CREATE NAMESPACE** | Isolated apps | Existing namespaces |
| **APPLY OUT OF SYNC ONLY** | Large apps (100+ resources) | Small apps, full sync needed |
| **SERVER-SIDE APPLY** | Large resources, conflicts | Simple apps |
| **REPLACE** | Immutable field changes | Normal operations (destructive) |
| **RETRY** | Temporary failures | Permanent errors |

---

### Testing Checklist

After completing all labs, you should understand:

- ✅ How auto-sync and self-heal prevent drift
- ✅ When pruning is safe and necessary
- ✅ How to handle schema validation issues
- ✅ Performance optimization with selective sync
- ✅ Conflict resolution strategies
- ✅ Resource dependency management

---

### Next Steps

1. Combine options for your use case
2. Test in non-production first
3. Document your team's standards
4. Create ArgoCD application templates
5. Implement proper RBAC and projects

---

## Quick Reference Card

Recommended production configuration:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/yourorg/yourrepo
    targetRevision: HEAD
    path: k8s/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
      - ServerSideApply=true
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

Good luck with your testing!
