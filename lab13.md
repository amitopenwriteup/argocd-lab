# ArgoCD ApplicationSet - Helm Charts Workshop

---

## Workshop Structure

```
┌──────────────────────────────────────────────────────────────┐
│              helm-appset-workshop/                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  01-basic-helm-app/                                          │
│       └── application.yaml                                  │
│  02-helm-appset-list/                                        │
│       └── applicationset.yaml                               │
│  03-helm-appset-git/                                         │
│       └── applicationset.yaml                               │
│  04-helm-appset-multi-env/                                   │
│       └── applicationset.yaml                               │
│  05-helm-appset-matrix/                                      │
│       └── applicationset.yaml                               │
│  06-helm-values/                                             │
│       ├── dev-values.yaml                                    │
│       ├── staging-values.yaml                                │
│       └── prod-values.yaml                                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Lab 1: Basic Helm Application

### File: 01-basic-helm-app/application.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-helm
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/bitnami/charts.git
    targetRevision: main
    path: bitnami/nginx
    helm:
      releaseName: my-nginx
      values: |
        replicaCount: 2
        service:
          type: ClusterIP
  destination:
    server: https://kubernetes.default.svc
    namespace: nginx-basic
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### Commands

```bash
# Create basic Helm application
kubectl apply -f basic-helm-app/application.yaml

# Check application status
argocd app get nginx-helm

# Sync application
argocd app sync nginx-helm

# View deployed resources
kubectl get all -n nginx-basic
```

---

## Lab 2: Helm ApplicationSet with List Generator

### File: 02-helm-appset-list/applicationset.yaml

### Commands

```bash
# Apply ApplicationSet
kubectl apply -f 02-helm-appset-list/applicationset.yaml

# Check ApplicationSet
kubectl get applicationset -n argocd

# List generated applications
kubectl get applications -n argocd
argocd app list

# Check specific application
argocd app get nginx-dev
argocd app get mysql-dev
argocd app get redis-dev

# View deployed resources
kubectl get all -n nginx-dev
kubectl get all -n mysql-dev
kubectl get all -n redis-dev
```

---

## Lab 3: Helm ApplicationSet with Git Generator

### Git Repository Structure

```
helm-values-repo/
├── apps/
│   ├── nginx/
│   │   ├── values.yaml
│   │   └── config.json
│   ├── apache/
│   │   ├── values.yaml
│   │   └── config.json
│   └── mysql/
│       ├── values.yaml
│       └── config.json
```

### File: apps/nginx/config.json

```json
{
  "chart": "nginx",
  "version": "15.14.0",
  "namespace": "nginx-git"
}
```

### File: apps/nginx/values.yaml

```yaml
replicaCount: 3
service:
  type: ClusterIP
  port: 80
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

### File: 03-helm-appset-git/applicationset.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: helm-git-appset
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/amitopenwriteup/helm-values-repo.git
      revision: HEAD
      directories:
      - path: apps/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        chart: '{{path.basename}}'
        repoURL: https://charts.bitnami.com/bitnami
        targetRevision: 15.14.0
        helm:
          releaseName: '{{path.basename}}'
          valueFiles:
          - '{{path}}/values.yaml'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}-ns'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

### Alternative: Git Files Generator with JSON

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: helm-git-files-appset
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/amitopenwriteup/helm-values-repo.git
      revision: HEAD
      files:
      - path: "apps/*/config.json"
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        chart: '{{chart}}'
        repoURL: https://charts.bitnami.com/bitnami
        targetRevision: '{{version}}'
        helm:
          releaseName: '{{path.basename}}'
          valueFiles:
          - 'apps/{{path.basename}}/values.yaml'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

### Commands

```bash
# Apply ApplicationSet
kubectl apply -f 03-helm-appset-git/applicationset.yaml

# Watch applications being created
watch kubectl get applications -n argocd

# Check generated apps
argocd app list

# Sync all apps
argocd app sync nginx
argocd app sync apache
argocd app sync mysql
```

---

## Lab 4: Multi-Environment Helm ApplicationSet

### File: 04-helm-appset-multi-env/applicationset.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: helm-multi-env
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - env: dev
        chart: nginx
        version: 15.14.0
        namespace: nginx-dev
        replicaCount: "1"
        serviceType: ClusterIP
        project: dev
      - env: staging
        chart: nginx
        version: 15.14.0
        namespace: nginx-staging
        replicaCount: "2"
        serviceType: ClusterIP
        project: staging
      - env: prod
        chart: nginx
        version: 15.14.0
        namespace: nginx-prod
        replicaCount: "3"
        serviceType: LoadBalancer
        project: production
  template:
    metadata:
      name: 'nginx-{{env}}'
      labels:
        environment: '{{env}}'
    spec:
      project: '{{project}}'
      source:
        chart: '{{chart}}'
        repoURL: https://charts.bitnami.com/bitnami
        targetRevision: '{{version}}'
        helm:
          releaseName: 'nginx-{{env}}'
          parameters:
          - name: replicaCount
            value: '{{replicaCount}}'
          - name: service.type
            value: '{{serviceType}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

```
┌──────────────────────────────────────────────────────────────┐
│              Multi-Env Applications Generated                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  nginx-dev      → namespace: nginx-dev     replicas: 1      │
│  nginx-staging  → namespace: nginx-staging replicas: 2      │
│  nginx-prod     → namespace: nginx-prod    replicas: 3      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Commands

```bash
# Apply multi-env ApplicationSet
kubectl apply -f 04-helm-appset-multi-env/applicationset.yaml

# List all applications
argocd app list

# Check each environment
argocd app get nginx-dev
argocd app get nginx-staging
argocd app get nginx-prod

# Verify deployments
kubectl get all -n nginx-dev
kubectl get all -n nginx-staging
kubectl get all -n nginx-prod

# Compare replica counts
kubectl get deployment -n nginx-dev nginx-dev -o yaml | grep replicas
kubectl get deployment -n nginx-staging nginx-staging -o yaml | grep replicas
kubectl get deployment -n nginx-prod nginx-prod -o yaml | grep replicas
```

---

## Lab 5: Matrix Generator for Helm Charts

### File: 05-helm-appset-matrix/applicationset.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: helm-matrix-appset
  namespace: argocd
spec:
  generators:
  - matrix:
      generators:
      - list:
          elements:
          - name: nginx
            chart: nginx
            version: 15.14.0
            port: "80"
          - name: apache
            chart: apache
            version: 10.7.0
            port: "8080"
      - list:
          elements:
          - env: dev
            namespace: dev-apps
            replicaCount: "1"
            project: dev
          - env: staging
            namespace: staging-apps
            replicaCount: "2"
            project: staging
          - env: prod
            namespace: prod-apps
            replicaCount: "3"
            project: production
  template:
    metadata:
      name: '{{name}}-{{env}}'
      labels:
        app: '{{name}}'
        environment: '{{env}}'
    spec:
      project: '{{project}}'
      source:
        chart: '{{chart}}'
        repoURL: https://charts.bitnami.com/bitnami
        targetRevision: '{{version}}'
        helm:
          releaseName: '{{name}}-{{env}}'
          parameters:
          - name: replicaCount
            value: '{{replicaCount}}'
          - name: service.port
            value: '{{port}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

```
┌──────────────────────────────────────────────────────────────┐
│        Matrix Generator: 2 apps × 3 envs = 6 apps           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  nginx-dev       nginx-staging       nginx-prod             │
│  apache-dev      apache-staging      apache-prod            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Commands

```bash
# Apply matrix ApplicationSet
kubectl apply -f 05-helm-appset-matrix/applicationset.yaml

# List all generated applications (6 total: 2 apps × 3 envs)
kubectl get applications -n argocd
argocd app list

# Check specific applications
argocd app get nginx-dev
argocd app get apache-prod

# Verify all deployments
kubectl get deployments --all-namespaces | grep -E 'nginx|apache'
```

---

## Lab 6: Advanced Helm Values Management

### File: 06-helm-values/dev-values.yaml

```yaml
replicaCount: 1
image:
  pullPolicy: Always
  tag: "latest"
service:
  type: ClusterIP
  port: 80
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
autoscaling:
  enabled: false
ingress:
  enabled: false
```

### File: 06-helm-values/staging-values.yaml

```yaml
replicaCount: 2
image:
  pullPolicy: IfNotPresent
  tag: "v1.0.0"
service:
  type: ClusterIP
  port: 80
resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 80
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: staging.example.com
      paths:
        - path: /
          pathType: Prefix
```

### File: 06-helm-values/prod-values.yaml

```yaml
replicaCount: 3
image:
  pullPolicy: IfNotPresent
  tag: "v1.0.0"
service:
  type: LoadBalancer
  port: 80
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: prod.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: prod-tls
      hosts:
        - prod.example.com
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - nginx
        topologyKey: kubernetes.io/hostname
```

### ApplicationSet Using Values Files

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: helm-with-values
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - env: dev
        valuesFile: dev-values.yaml
        namespace: nginx-dev
      - env: staging
        valuesFile: staging-values.yaml
        namespace: nginx-staging
      - env: prod
        valuesFile: prod-values.yaml
        namespace: nginx-prod
  template:
    metadata:
      name: 'nginx-{{env}}'
    spec:
      project: default
      source:
        chart: nginx
        repoURL: https://charts.bitnami.com/bitnami
        targetRevision: 15.14.0
        helm:
          releaseName: 'nginx-{{env}}'
          valueFiles:
          - 'https://raw.githubusercontent.com/amitopenwriteup/helm-values/main/{{valuesFile}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

---

## Common Patterns

### Pattern 1: Multi-Cluster with Cluster Generator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: helm-multi-cluster
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: production
  template:
    metadata:
      name: 'nginx-{{name}}'
    spec:
      project: default
      source:
        chart: nginx
        repoURL: https://charts.bitnami.com/bitnami
        targetRevision: 15.14.0
        helm:
          releaseName: nginx
          parameters:
          - name: replicaCount
            value: "3"
      destination:
        server: '{{server}}'
        namespace: nginx
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### Pattern 2: Pull Request Generator for Preview Environments

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: helm-pr-preview
  namespace: argocd
spec:
  generators:
  - pullRequest:
      github:
        owner: amitopenwriteup
        repo: my-app
        tokenRef:
          secretName: github-token
          key: token
      requeueAfterSeconds: 300
  template:
    metadata:
      name: 'preview-pr-{{number}}'
    spec:
      project: default
      source:
        chart: my-app
        repoURL: https://charts.example.com
        targetRevision: latest
        helm:
          releaseName: 'pr-{{number}}'
          parameters:
          - name: image.tag
            value: 'pr-{{number}}'
          - name: ingress.hosts[0]
            value: 'pr-{{number}}.preview.example.com'
      destination:
        server: https://kubernetes.default.svc
        namespace: 'preview-pr-{{number}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

---

## Complete Workflow Commands

```bash
# 1. Verify ArgoCD is running
kubectl get pods -n argocd

# 2. Create dev project
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: dev
  namespace: argocd
spec:
  destinations:
  - namespace: '*'
    server: https://kubernetes.default.svc
  sourceRepos:
  - '*'
EOF

# 3. Apply basic Helm application
kubectl apply -f 01-basic-helm-app/application.yaml

# 4. Apply Helm ApplicationSet with list generator
kubectl apply -f 02-helm-appset-list/applicationset.yaml

# 5. Apply multi-environment ApplicationSet
kubectl apply -f 04-helm-appset-multi-env/applicationset.yaml

# 6. Apply matrix generator ApplicationSet
kubectl apply -f 05-helm-appset-matrix/applicationset.yaml

# 7. Check all ApplicationSets
kubectl get applicationset -n argocd

# 8. List all applications
kubectl get applications -n argocd
argocd app list

# 9. Check specific application
argocd app get nginx-dev

# 10. Sync by label
argocd app sync --selector environment=dev
argocd app sync --selector environment=staging
argocd app sync --selector environment=prod

# 11. Verify deployments
kubectl get deployments --all-namespaces | grep nginx

# 12. Check Helm releases
helm list -A

# 13. Clean up
kubectl delete applicationset helm-appset-list -n argocd
kubectl delete applicationset helm-multi-env -n argocd
kubectl delete applicationset helm-matrix-appset -n argocd
kubectl delete application nginx-helm -n argocd
```

---

## Troubleshooting Commands

```bash
# Check ApplicationSet status
kubectl describe applicationset helm-appset-list -n argocd

# Check ApplicationSet controller logs
kubectl logs -n argocd \
  -l app.kubernetes.io/name=argocd-applicationset-controller \
  --tail=100

# Check application sync status
argocd app get nginx-dev

# Check Helm release values
helm get values nginx-dev -n nginx-dev

# Check Helm chart manifests
argocd app manifests nginx-dev | grep -A 10 "kind: Deployment"

# Manually sync application
argocd app sync nginx-dev --force

# Refresh application
argocd app refresh nginx-dev

# View application diff
argocd app diff nginx-dev

# Check events
kubectl get events -n nginx-dev --sort-by='.lastTimestamp'
```

---

## Quick Reference

| Command | Description |
|---|---|
| `kubectl apply -f applicationset.yaml` | Create ApplicationSet |
| `kubectl get applicationset -n argocd` | List ApplicationSets |
| `argocd app list` | List all applications |
| `argocd app get <name>` | Get app details |
| `argocd app sync <name>` | Sync application |
| `argocd app sync --selector env=dev` | Sync by label |
| `argocd app refresh <name>` | Refresh from Git/Helm |
| `argocd app diff <name>` | Show diff |
| `helm list -A` | List all Helm releases |
| `helm get values <name> -n <ns>` | Get Helm values |
| `kubectl delete applicationset <name> -n argocd` | Delete ApplicationSet |
