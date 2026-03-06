# ArgoCD Projects — Labs & CLI Reference
**Kind Cluster | kubectl | GitHub | GitOps**  
*End-to-End Setup & Application Deployment*  
*February 2026*

---

## Table of Contents

1. [List & Delete Applications](#0-list--delete-applications)
2. [Lab 1: Understanding the Default Project](#lab-1-understanding-the-default-project)
3. [Lab 2: Create a Basic Project](#lab-2-create-a-basic-project)
4. [Add Destinations](#add-destinations)
5. [Add Source Repositories](#add-source-repositories)
6. [Manage Project Roles](#manage-project-roles)
7. [View and Update Projects](#view-and-update-projects)

---

## 0. List & Delete Applications

Before starting the labs, list existing applications and clean up as needed:

```bash
# List all applications
argocd app list

# Delete an application
argocd app delete <app-name>
```

---

## Lab 1: Understanding the Default Project

```bash
# View the default project
argocd proj list
kubectl get appproject -n argocd

# Describe the default project
kubectl get appproject default -n argocd -o yaml

# List projects using ArgoCD CLI
argocd proj list
```

---

## Lab 2: Create a Basic Project

### Create the manifest file

```bash
vi dev-project.yaml
```

Paste the following content:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: dev-project
  namespace: argocd
spec:
  description: Development environment project

  # Source repositories allowed
  sourceRepos:
  - '*'

  # Destination clusters and namespaces
  destinations:
  - namespace: 'dev-*'
    server: https://kubernetes.default.svc
  - namespace: development
    server: https://kubernetes.default.svc

  # Cluster resource whitelist (what can be deployed)
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace

  # Namespace resource whitelist
  namespaceResourceWhitelist:
  - group: 'apps'
    kind: Deployment
  - group: ''
    kind: Service
  - group: ''
    kind: ConfigMap
  - group: ''
    kind: Secret
```

### Apply & Verify

```bash
# Create the project
kubectl apply -f dev-project.yaml

# Verify project creation
kubectl get appproject -n argocd

# Get project details
kubectl get appproject dev-project -n argocd -o yaml

# Using ArgoCD CLI
argocd proj get dev-project

# List all projects
argocd proj list
```

### Create Project via CLI (Alternative)

```bash
argocd proj create dev \
  --description "Secure project" \
  --dest https://kubernetes.default.svc,secure-* \
  --src https://github.com/amitopenwriteup/* \
  --allow-cluster-resource Namespace \
  --allow-namespaced-resource apps/Deployment \
  --allow-namespaced-resource core/Service
```

---

## Add Destinations

```bash
# Add a specific destination to an existing project
argocd proj add-destination dev https://kubernetes.default.svc default

# Add a wildcard destination
argocd proj add-destination dev https://kubernetes.default.svc '*'
```

---

## Add Source Repositories

```bash
# Add a specific source repo
argocd proj add-source dev \
  https://github.com/amitopenwriteup/argocdlistgit.git

# Add a wildcard source
argocd proj add-source dev 'https://github.com/amitopenwriteup/*'
```

---

## Manage Project Roles

```bash
# Create a role
argocd proj role create dev developer

# Add a policy to the role
argocd proj role add-policy dev developer \
  --action get \
  --permission allow \
  --object 'applications'

# Add a group to the role
argocd proj role add-group dev developer my-sso-group

# List roles
argocd proj role list dev

# Get role details
argocd proj role get dev developer
```

---

## View and Update Projects

```bash
# List all projects
argocd proj list

# Get project details
argocd proj get dev

# Get project in YAML format
argocd proj get dev -o yaml

# Edit project directly
kubectl edit appproject dev -n argocd

# Delete a project
argocd proj delete dev
# Or
kubectl delete appproject dev -n argocd
```
