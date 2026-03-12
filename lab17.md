# ArgoCD ApplicationSet — Helm Chart List Generator with Sync Waves


---

## 1. Overview

This ApplicationSet uses the **List Generator** to deploy multiple Helm charts from the Bitnami registry to an Amazon EKS cluster. Each list element defines a self-contained Helm release with its own chart name, version, and **sync wave** — controlling the order in which applications are deployed.

**Key capabilities in this configuration:**

- Helm charts sourced directly from a public Helm registry (not a Git repo)
- Per-application sync wave ordering via ArgoCD annotations
- Automated pruning and self-healing across all releases
- All releases land in a single shared namespace `amit-default`

---

## 2. YAML Manifest

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
            syncWave: "1"          # nginx deploys first
          - chart: mariadb
            releaseName: my-mysql
            targetRevision: 25.0.1
            syncWave: "2"          # mysql deploys second
  template:
    metadata:
      name: '{{releaseName}}'
      annotations:
        argocd.argoproj.io/sync-wave: '{{syncWave}}'   # controls deploy order
    spec:
      project: stage-devseccops
      source:
        repoURL: https://charts.bitnami.com/bitnami
        chart: '{{chart}}'
        targetRevision: '{{targetRevision}}'
        helm:
          releaseName: '{{releaseName}}'
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

## 3. Generated Applications

The list generator produces one ArgoCD Application per element:

| Application Name | Chart | Version | Namespace | Sync Wave |
|---|---|---|---|---|
| `my-nginx` | `nginx` | `18.1.2` | `amit-default` | `1` |
| `my-mysql` | `mariadb` | `25.0.1` | `amit-default` | `2` |

---

## 4. Template Variables

| Variable | Source Element Key | Example Value |
|---|---|---|
| `{{chart}}` | `chart` | `nginx` |
| `{{releaseName}}` | `releaseName` | `my-nginx` |
| `{{targetRevision}}` | `targetRevision` | `18.1.2` |
| `{{syncWave}}` | `syncWave` | `"1"` |

> All keys defined under each list element are directly available as template variables.

---

## 5. Field Reference

| Field | Value | Description |
|---|---|---|
| `generators[].list.elements` | Array of objects | Each element renders one Application |
| `chart` | `nginx` / `mysql` | Helm chart name from Bitnami registry |
| `releaseName` | `my-nginx` / `my-mysql` | Helm release name and ArgoCD Application name |
| `syncWave` | `"1"` / `"2"` | Deployment order (lower = earlier) |
| `source.repoURL` | `https://charts.bitnami.com/bitnami` | Bitnami public Helm registry |
| `source.chart` | `{{chart}}` | Chart name resolved from list element |
| `helm.releaseName` | `{{releaseName}}` | Helm release name inside the cluster |
| `destination.name` | EKS cluster ARN | Target EKS cluster identifier |
| `destination.namespace` | `amit-default` | All releases deploy to this namespace |
| `automated.prune` | `true` | Removes Helm resources deleted from config |
| `automated.selfHeal` | `true` | Reverts manual Helm/kubectl changes |
| `CreateNamespace=true` | syncOption | Creates `amit-default` if it does not exist |

---

## 6. Sync Waves — Deployment Ordering

Sync waves control the **sequence** in which ArgoCD syncs resources during a single sync operation. Lower wave numbers are applied first; the next wave only begins after all resources in the current wave are healthy.

```
Sync Wave 1                    Sync Wave 2
───────────────                ───────────────
my-nginx (nginx 18.1.2)   →   my-mysql (mysql 11.2.5)
  Deploy & wait healthy          Deploy after nginx is ready
```

### How Sync Waves Work

1. ArgoCD groups all Applications by their `sync-wave` annotation value
2. Wave `1` resources are synced and ArgoCD waits for them to become **Healthy**
3. Once wave `1` is healthy, wave `2` begins
4. This continues until all waves complete

### Wave Annotation

```yaml
annotations:
  argocd.argoproj.io/sync-wave: '{{syncWave}}'
```

> **Note:** `syncWave` values are strings in the list elements (`"1"`, `"2"`) but ArgoCD interprets them as integers for ordering. Negative values (e.g. `"-1"`) are valid and deploy before wave `0`.



```bash
kubectl apply -f applicationset.yaml -n argocd
```

### Verify Applications Were Created

```bash
# List generated Applications
argocd app list

# Check sync status of each release
argocd app get my-nginx
argocd app get my-mysql

# Watch live sync progress
argocd app list --output wide

# Verify Helm releases on the cluster
kubectl get pods -n amit-default
helm list -n amit-default
```

---

## 10. Upgrading a Chart Version

To upgrade a chart, update the `targetRevision` in the list element:

```yaml
- chart: nginx
  releaseName: my-nginx
  targetRevision: 18.2.0    # ← bump version here
  syncWave: "1"
```

Commit and push — ArgoCD will detect the drift and automatically upgrade the Helm release with `selfHeal: true` active.

---

## 11. Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| `cluster not found` error | EKS cluster not registered | Run `argocd cluster add` (see Section 8) |
| `chart not found` in registry | Wrong chart name or version | Verify with `helm search repo bitnami/<chart>` |
| Apps created but not synced | Project permission denied | Check source/destination allow rules in project |
| Wave 2 starts before Wave 1 healthy | Health check not passing | Inspect `argocd app get my-nginx` for health status |
| Namespace not created | Missing `CreateNamespace=true` | Add to `syncOptions` |
| Helm release name conflict | Duplicate `releaseName` in elements | Ensure each `releaseName` is unique |

---

*This pattern is well suited for deploying ordered stacks of third-party Helm charts — such as ingress controllers, monitoring agents, and application services — with guaranteed dependency sequencing via sync waves.*
