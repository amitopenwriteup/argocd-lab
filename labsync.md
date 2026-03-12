# ArgoCD Sync Policy — Lab 3: Sync Options Testing
**Kind Cluster | kubectl | GitHub | GitOps**  
*End-to-End Setup & Application Deployment*  
*February 2026*

---

## Objective

Test **Skip Schema Validation**, **Prune Last**, **Respect Ignore Differences**, and **Replace** sync options.

---

## Create the Application in ArgoCD UI

**1.** Click **+ NEW APP**

**2.** Fill in **General**:

| Field | Value |
|-------|-------|
| **Application Name** | `lab3-sync-options` |
| **Project** | `myproj` |
| **Sync Policy** | Manual |

**3.** Fill in **Source**:

| Field | Value |
|-------|-------|
| **Repository URL** | `https://github.com/amitopenwriteup/argoappsynclab.git` |
| **Revision** | `HEAD` |
| **Path** | `lab3-sync-options` |

**4.** Fill in **Destination**:

| Field | Value |
|-------|-------|
| **Cluster URL** | `https://kubernetes.default.svc` |
| **Namespace** | `lab3` |

**5.** Configure **Sync Options** (all unchecked initially except Auto-Create Namespace):

| Option | Setting |
|--------|---------|
| ⬜ Skip Schema Validation | Unchecked |
| ⬜ Prune Last | Unchecked |
| ⬜ Respect Ignore Differences | Unchecked |
| ✅ Auto-Create Namespace | Checked |
| ⬜ Apply Out of Sync Only | Unchecked |
| ⬜ Server-Side Apply | Unchecked |
| ⬜ Replace | Unchecked |

**6.** Click **Create**, then click **Sync → Synchronize**.

---




### Test 2 — Prune Last
go to git repo which you cloned

```
cd argoappsynclab/lab3-sync-options/templates

```

```bash

# Create a ConfigMap
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
```

Sync the application in the ArgoCD UI, then create a deployment that depends on the ConfigMap:

```bash
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

**Observe in the ArgoCD UI:**

- Enable ✅ **Prune Resources** in sync policy
- Enable ✅ **Prune Last** in sync options
- Click **Sync**
- With Prune Last, the dependent deployment is removed before the ConfigMap, preventing missing dependency errors

---

### Test 3 — Respect Ignore Differences

In the ArgoCD UI, configure Ignore Differences:

1. Click the app → **App Details → Edit**
2. Switch to **YAML mode** and add the following:

```yaml
ignoreDifferences:
- group: apps
  kind: Deployment
  jsonPointers:
  - /spec/replicas
```

Manually scale the deployment to create drift:

```bash
kubectl scale deployment demo-app -n lab3 --replicas=5
```

**Observe in the ArgoCD UI:**

- Application shows **OutOfSync**
- Enable ✅ **Respect Ignore Differences** in sync options
- Click **Sync**
- The replica count difference is now ignored — app shows as **Synced** without reverting to 2 replicas

> ℹ️ Without this option, ArgoCD would sync the replicas back to the Git-defined value on every sync.

---

### Test 4 — Replace Option

Create a StatefulSet with a `serviceName` field (immutable):

```bash
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
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
EOF

git add statefulset.yaml
git commit -m "Add statefulset"
git push
```

Sync the application first, then modify the immutable field:

```bash
sed -i 's/serviceName: "nginx"/serviceName: "nginx-service"/' statefulset.yaml
git add statefulset.yaml
git commit -m "Modify immutable field"
git push
```

**Observe in the ArgoCD UI:**

- Sync **without** Replace → error (immutable field cannot be updated)
- Enable ✅ **Replace** in sync options and sync again
- The StatefulSet is deleted and recreated with the new value

> ⚠️ **Warning:** Replace is destructive — use only when immutable field changes are required.

---

## Cleanup

In the ArgoCD UI: click **lab3-sync-options → Delete → Confirm**.
