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
cat > service1.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx
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

```
```
git add service1.yaml statefulset.yaml
git commit -m "Add statefulset and stateless service"
git push
```

```

**UI option sync it**


# Sync first, then modify immutable field
sed -i 's/serviceName: "nginx"/serviceName: "nginx-service"/' statefulset.yaml
git add statefulset.yaml
git commit -m "Modify immutable field"
git push
```

**Observe in UI:**
- Try to **SYNC** without REPLACE — you'll get an error (immutable field)
- Enable **REPLACE** and **FORCE** option
- Click **SYNC** — resource will be deleted and recreated

---

### Cleanup

Delete application from UI.

---

