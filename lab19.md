# ArgoCD & Argo Workflows Integration Workshop
### Building Complete CI/CD Pipelines with GitOps

| Field | Detail |
|-------|--------|
| **Duration** | 4–5 hours (hands-on) |
| **Level** | Intermediate |
| **Prerequisites** | Kubernetes basics, Git, Docker |

---

## Table of Contents

1. [Workshop Overview & Objectives](#1-workshop-overview--objectives)
2. [Environment Setup](#2-environment-setup)
3. [Module 1: ArgoCD Fundamentals](#3-module-1-argocd-fundamentals)
4. [Module 2: Argo Workflows Fundamentals](#4-module-2-argo-workflows-fundamentals)
5. [Module 3: Integration Patterns](#5-module-3-integration-patterns)
6. [Module 4: Complete CI/CD Pipeline](#6-module-4-complete-cicd-pipeline)
7. [Module 5: Advanced Topics](#7-module-5-advanced-topics)
8. [Troubleshooting & Best Practices](#8-troubleshooting--best-practices)
9. [Additional Resources](#9-additional-resources)

---

## 1. Workshop Overview & Objectives

### Learning Objectives

By the end of this workshop, you will be able to:

- Understand the architecture and purpose of both ArgoCD and Argo Workflows
- Deploy and configure ArgoCD for GitOps continuous delivery
- Create and execute complex workflows using Argo Workflows
- Integrate both tools to build complete CI/CD pipelines
- Implement GitOps best practices in production environments
- Troubleshoot common issues and optimize workflow performance

### Architecture Overview

The integration between ArgoCD and Argo Workflows creates a powerful GitOps-based CI/CD ecosystem:

| Component | Purpose | Key Features |
|-----------|---------|--------------|
| **ArgoCD** | GitOps continuous delivery | Auto-sync, rollback, multi-cluster support |
| **Argo Workflows** | Workflow orchestration & CI | DAGs, parallel execution, artifacts |

---

## 2. Environment Setup

### Prerequisites

Ensure you have the following installed:

- Kubernetes cluster (v1.24+) — minikube, kind, or cloud provider
- `kubectl` CLI tool
- Docker installed and running
- Git CLI
- Helm 3.x
- Text editor (VS Code recommended)

---

### Installing ArgoCD

**Step 1: Create namespace and install ArgoCD**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Step 2: Expose ArgoCD UI**

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

**Step 3: Get initial admin password**

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

> Access the UI at **https://localhost:8080** (username: `admin`)

---

### Installing Argo Workflows

**Step 1: Create namespace and install Argo Workflows**

```bash
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/latest/download/install.yaml
```

**Step 2: Expose Argo Workflows UI**

```bash
kubectl -n argo port-forward deployment/argo-server 2746:2746 \
  --address 0.0.0.0 >/tmp/argo-portforward.log 2>&1 &

ufw status
sudo ufw allow 2746/tcp
sudo ufw reload
```

> Access the UI at **https://localhost:2746**

---

### Install Argo CLI

```bash
# Linux/Mac
curl -sLO https://github.com/argoproj/argo-workflows/releases/latest/download/argo-linux-amd64.gz
gunzip argo-linux-amd64.gz
chmod +x argo-linux-amd64
sudo mv ./argo-linux-amd64 /usr/local/bin/argo
```

---

## 3. Module 1: ArgoCD Fundamentals

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Application** | A group of Kubernetes resources defined by a manifest |
| **Project** | Logical grouping of applications with access controls |
| **Sync Status** | Comparison between Git state and cluster state |
| **Health Status** | Runtime health of deployed resources |

---

### Exercise 1: Deploy Your First Application

**Step 1: Fork the sample repository**

```bash
git clone https://github.com/argoproj/argocd-example-apps.git
cd argocd-example-apps/guestbook
```

**Step 2: Create an ArgoCD application**

```yaml
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

**Step 3: Watch the deployment in ArgoCD UI**

- Navigate to the **Applications** tab
- Click on the **guestbook** application
- Observe the sync status and resource tree

---

### Exercise 2: GitOps Workflow

**Step 1: Modify the application in Git**

```bash
# Edit guestbook/guestbook-ui-deployment.yaml
# Change replicas from 1 to 3
# Commit and push changes
```

**Step 2: Watch ArgoCD automatically sync the changes**

- Due to automated sync policy, changes deploy automatically
- Verify with: `kubectl get pods -n default`

---

## 4. Module 2: Argo Workflows Fundamentals

### Workflow Anatomy

A workflow consists of:

- **Metadata** — name, namespace, labels
- **Spec** — entrypoint and template definitions
- **Templates** — steps, DAG, container, script, resource

---

### Exercise 3: Create a Simple Workflow

> ⚠️ **Common RBAC Issue**
>
> If your workflow fails with **exit code 64**, it is likely because the service account
> `system:serviceaccount:argo:default` does not have permission to create
> `workflowtaskresults.argoproj.io`. Apply the fix below before submitting workflows.

**Quick Fix — Grant RBAC permissions (lab/training environments)**

```bash
kubectl create clusterrolebinding argo-default-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=argo:default
```

**Create `hello-world.yaml`**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-
  namespace: argo
spec:
  entrypoint: whalesay
  templates:
    - name: whalesay
      container:
        image: docker/whalesay
        command: ["cowsay"]
        args: ["Hello ArgoCD Workshop!"]
```

```bash
argo submit -n argo hello-world.yaml --watch
```

---

### Exercise 4: DAG Workflow

Create a multi-step workflow with dependencies — save as `dag-workflow.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: dag-workflow-
  namespace: argo
spec:
  entrypoint: build-test-deploy
  templates:
    - name: build-test-deploy
      dag:
        tasks:
          - name: build
            template: echo
            arguments:
              parameters:
                - name: message
                  value: "Building application"

          - name: test
            dependencies: [build]
            template: echo
            arguments:
              parameters:
                - name: message
                  value: "Running tests"

          - name: deploy
            dependencies: [test]
            template: echo
            arguments:
              parameters:
                - name: message
                  value: "Deploying to cluster"

    - name: echo
      inputs:
        parameters:
          - name: message
      container:
        image: alpine:latest
        command: ["echo"]
        args: ["{{inputs.parameters.message}}"]
```

```bash
argo submit -n argo dag-workflow.yaml --watch
```

---

## 5. Module 3: Integration Patterns

### Pattern 1: Build → Update Manifest → Auto-Deploy

This is the most common integration pattern:

1. Argo Workflow builds and pushes a container image
2. Workflow updates the Kubernetes manifest in Git with the new image tag
3. ArgoCD detects the Git change and deploys automatically

---

### Exercise 5: Complete CI/CD Workflow

**Step 1: Create Docker Hub secret**

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-username=<your-username> \
  --docker-password=<your-token> \
  --docker-email=<your-email> \
  -n argo
```

**Step 2: Build image with Kaniko**

```yaml
- name: build-image
  container:
    image: gcr.io/kaniko-project/executor:latest
    args:
      - --dockerfile=/work/source/Dockerfile
      - --context=/work/source
      - --destination=docker.io/<your-username>/cicdproject:{{workflow.uid}}
      - --no-push
    volumeMounts:
      - name: workdir
        mountPath: /work
      - name: docker-config
        mountPath: /kaniko/.docker
```

---

### Pattern 2: Workflow Triggers ArgoCD Sync

For more control, workflows can explicitly trigger ArgoCD:

```yaml
- name: trigger-argocd
  container:
    image: argoproj/argocd:latest
    command: [sh, -c]
    args:
      - |
        argocd app sync myapp --grpc-web
```

---

### Pattern 3: ArgoCD Manages Workflows

Deploy workflow definitions via ArgoCD for GitOps workflow management:

- Store `WorkflowTemplates` in Git
- ArgoCD deploys them to the cluster
- Developers submit workflows using these templates

---

## 6. Module 4: Complete CI/CD Pipeline

### Real-World Example: Microservice Deployment

A complete pipeline includes:

- Code checkout from Git
- Unit testing
- Container image building
- Security scanning
- Image pushing to registry
- Manifest update in Git
- Automated deployment via ArgoCD
- Slack notifications

### Pipeline Stages

| Stage | Tool | Purpose |
|-------|------|---------|
| **CI** | Argo Workflows | Build, test, scan, push |
| **GitOps** | Git Repository | Source of truth for manifests |
| **CD** | ArgoCD | Deploy to Kubernetes |

---

## 7. Module 5: Advanced Topics

### Multi-Environment Deployments

Use ArgoCD `ApplicationSets` for deploying to multiple environments:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservice
spec:
  generators:
    - list:
        elements:
          - env: dev
            cluster: dev-cluster
          - env: staging
            cluster: staging-cluster
          - env: prod
            cluster: prod-cluster
  template:
    metadata:
      name: 'myapp-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/manifests
        path: 'overlays/{{env}}'
      destination:
        server: '{{cluster}}'
        namespace: myapp
```

---

### Rollback Strategies

ArgoCD provides multiple rollback options:

- Manual rollback via UI or CLI
- Git revert and auto-sync
- Integration with Argo Rollouts for progressive delivery

---

### Secrets Management

Best practices for handling secrets:

- Use **Sealed Secrets** or **External Secrets Operator**
- Integrate with **HashiCorp Vault**
- Use cloud provider secret managers (AWS Secrets Manager, GCP Secret Manager)

---

### Monitoring and Observability

| Component | Metrics |
|-----------|---------|
| **ArgoCD** | Sync status, health status, sync frequency |
| **Argo Workflows** | Workflow success rate, duration, resource usage |
| **Applications** | Deployment frequency, lead time, MTTR |

---

## 8. Troubleshooting & Best Practices

### Common Issues

#### ArgoCD

| Issue | Cause | Solution |
|-------|-------|----------|
| App stuck syncing | Resource hooks failing | Check hook logs, fix errors |
| Out of sync | Manual cluster changes | Enable `selfHeal` or manual sync |
| Permission denied | RBAC misconfiguration | Update project/cluster RBAC |

#### Argo Workflows

| Issue | Cause | Solution |
|-------|-------|----------|
| Workflow pending | Insufficient resources | Scale cluster or adjust limits |
| Image pull errors | Registry auth issues | Configure `imagePullSecrets` |
| Artifact errors | S3 misconfiguration | Verify artifact repository config |

---

### Best Practices

#### Repository Structure

```
manifests/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   ├── staging/
│   └── prod/
workflows/
├── ci-pipeline.yaml
└── templates/
```

#### Security Best Practices

- Use RBAC to limit workflow permissions
- Never commit secrets to Git repositories
- Use service accounts with minimal permissions
- Enable ArgoCD SSO and audit logging
- Scan container images for vulnerabilities

#### Performance Optimization

- Use workflow templates to reduce duplication
- Implement caching for build dependencies
- Parallelize independent workflow steps
- Configure resource limits appropriately
- Use ArgoCD sync waves for ordered deployments

---

## 9. Additional Resources

### Official Documentation

| Resource | URL |
|----------|-----|
| ArgoCD Documentation | https://argo-cd.readthedocs.io |
| Argo Workflows Documentation | https://argoproj.github.io/workflows |
| Argo Project GitHub | https://github.com/argoproj |

### Community Resources

- **CNCF Slack** — `#argo-cd` and `#argo-workflows` channels
- **Argo Community Meetings** — Bi-weekly on Zoom
- **GitHub Discussions** for Q&A

### Example Projects

- [Argo Example Apps](https://github.com/argoproj/argocd-example-apps)
- [Argo Workflow Examples](https://github.com/argoproj/argo-workflows/tree/master/examples)

---

### Next Steps

After completing this workshop, consider:

1. Implementing this pipeline in your organization
2. Exploring **Argo Rollouts** for progressive delivery
3. Setting up multi-cluster deployments
4. Integrating with your monitoring and observability stack
5. Contributing to the Argo Project community

---

> 🎉 **Thank You!**
> *Questions & Feedback welcome — Happy GitOps-ing!*
