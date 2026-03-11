# ArgoCD ApplicationSet — `helm-application-amit`

> **Resource:** `helm-application-amit` | **Namespace:** `argocd` | **Service Type:** `ClusterIP`

---

## 1. Overview

This document describes the ArgoCD `ApplicationSet` resource `helm-application-amit`, which automates the deployment of multiple Helm chart applications to a Kubernetes cluster using the Bitnami Helm registry. All deployed services are configured with `ClusterIP` as the service type.

| Property | Value |
|---|---|
| API Version | `argoproj.io/v1alpha1` |
| Kind | `ApplicationSet` |
| Resource Name | `helm-application-amit` |
| ArgoCD Namespace | `argocd` |
| Target Cluster | `https://kubernetes.default.svc` |
| Target Namespace | `amit-default` |
| Chart Registry | `https://charts.bitnami.com/bitnami` |
| Service Type | `ClusterIP` |

---

## 2. Generator Configuration

### 2.1 List Elements

| Field | Value | Description |
|---|---|---|
| `chart` | `nginx` | Bitnami Helm chart name |
| `releaseName` | `my-nginx` | Helm release and ArgoCD app name |
| `targetRevision` | `22.4.3` | Nginx chart version (latest) |
| `chart` | `apache` | Bitnami Helm chart name |
| `releaseName` | `my-apache` | Helm release and ArgoCD app name |
| `targetRevision` | `11.2.5` | Apache chart version |

---

## 3. Helm Values — Service Type

The `service.type` is set to `ClusterIP` via `helm.values` in the template. This applies to **both** nginx and apache releases, making the services only accessible within the cluster (no external IP assigned).

| Setting | Value | Effect |
|---|---|---|
| `service.type` | `ClusterIP` | Internal cluster access only |

> **When to use ClusterIP:** Suitable when the service is consumed by other pods inside the cluster (e.g., via an Ingress controller or internal microservices), rather than being exposed directly to external traffic.

---

## 4. Sync Policy

| Setting | Behavior |
|---|---|
| `automated.prune: true` | Removes resources no longer present in the Helm chart |
| `automated.selfHeal: true` | Auto-reverts manual cluster changes (drift correction) |
| `CreateNamespace=true` | Auto-creates `amit-default` namespace if it does not exist |

---

## 5. Full YAML Manifest
```
vi helmset.yaml
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: helm-application-amit
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - chart: nginx
            releaseName: my-nginx
            targetRevision: 22.4.3
          - chart: mariadb
            releaseName: my-mysql
            targetRevision: 25.0.1
  template:
    metadata:
      name: '{{releaseName}}'
    spec:
      project: default
      source:
        repoURL: https://charts.bitnami.com/bitnami
        chart: '{{chart}}'
        targetRevision: '{{targetRevision}}'
        helm:
          releaseName: '{{releaseName}}'
          values: |
            service:
              type: ClusterIP
      destination:
        server: https://kubernetes.default.svc
        namespace: amit-default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

---

## 6. Verify Deployment

```bash
# ApplicationSet Commands

kubectl create -f helmset.yaml
kubectl get applicationset -n argocd
kubectl get applicationset helm-application-amit -n argocd
kubectl describe applicationset helm-application-amit -n argocd
kubectl get applications -n argocd
kubectl get application my-nginx -n argocd
kubectl get application my-mysql -n argocd
kubectl describe application my-nginx -n argocd
kubectl describe application my-mysql -n argocd
kubectl delete applicationset helm-application-amit -n argocd

```

---

*Generated: March 2026 | Version: 1.1 | Nginx Chart: `22.4.3` | Service Type: `ClusterIP`*
