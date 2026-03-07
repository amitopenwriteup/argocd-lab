# ArgoCD Lab 12 — Resource Hooks

> **Topic:** Sync lifecycle hooks | **Annotation:** `argocd.argoproj.io/hook` | **Use Cases:** DB migrations, smoke tests, Slack notifications

---

## 1. Overview

Resource Hooks allow you to run custom logic at specific **phases of an ArgoCD sync operation**. They are standard Kubernetes manifests (typically `Job` or `Pod`) annotated with `argocd.argoproj.io/hook` to tell ArgoCD when to execute them.

**Common use cases:**
- Run database schema migrations before deploying new app versions (`PreSync`)
- Execute smoke/integration tests after deployment completes (`PostSync`)
- Send Slack or webhook notifications on success or failure (`PostSync` / `SyncFail`)
- Seed configuration data or run one-time setup tasks

---

## 2. Sync Phases & Hook Types

ArgoCD defines five hook phases that map to the sync lifecycle:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   PreSync    │────▶│     Sync     │────▶│   PostSync   │────▶│    Skip      │
│              │     │  (manifests) │     │              │     │  (no apply)  │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                                                                 ┌──────────────┐
                                                   on failure ──▶│  SyncFail    │
                                                                 └──────────────┘
```

| Hook Type | Runs When | Typical Use |
|---|---|---|
| `PreSync` | Before any manifests are applied | DB migrations, pre-flight checks |
| `Sync` | At the same time as manifests are applied | Parallel setup tasks |
| `PostSync` | After all resources are Healthy | Integration tests, notifications |
| `SyncFail` | When the sync operation fails | Alert on failure, rollback triggers |
| `Skip` | Never applied | Exclude a resource from sync |

---

## 3. Basic Hook Structure

Any Kubernetes resource can be a hook — add the annotation to activate it:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: schema-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: my-app:latest
          command: ["./migrate.sh"]
      restartPolicy: Never
  backoffLimit: 0
```

> Multiple hooks can be specified as a comma-separated list:
> `argocd.argoproj.io/hook: PreSync,Sync`

---

## 4. Hook Manifests

### 4.1 PreSync — Database Migration

Runs **before** application manifests are applied. ArgoCD waits for this Job to succeed before proceeding.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: schema-migrate-
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: db-migrate
          image: bitnami/postgresql:15
          command:
            - "/bin/sh"
            - "-c"
            - "psql $DATABASE_URL -f /migrations/schema.sql"
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
      restartPolicy: Never
  backoffLimit: 2
```

### 4.2 PostSync — Smoke Test

Runs **after** all resources are synced and healthy.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: smoke-test-
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: smoke-test
          image: curlimages/curl:latest
          command:
            - "sh"
            - "-c"
            - |
              curl -sf http://my-app-service/healthz || exit 1
              echo "Smoke test passed"
      restartPolicy: Never
  backoffLimit: 1
```

### 4.3 PostSync — Slack Success Notification

Send a Slack message when sync completes successfully.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: notify-success-
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: slack-notify
          image: curlimages/curl:latest
          command:
            - "curl"
            - "-X"
            - "POST"
            - "--data-urlencode"
            - >
              payload={"channel": "#deployments", "username": "ArgoCD",
              "text": ":white_check_mark: App sync succeeded!", "icon_emoji": ":rocket:"}
            - "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
      restartPolicy: Never
  backoffLimit: 2
```

### 4.4 SyncFail — Slack Failure Notification

Runs **only when the sync fails** — ideal for alerting on broken deployments.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: notify-failure-
  annotations:
    argocd.argoproj.io/hook: SyncFail
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: slack-notify
          image: curlimages/curl:latest
          command:
            - "curl"
            - "-X"
            - "POST"
            - "--data-urlencode"
            - >
              payload={"channel": "#deployments", "username": "ArgoCD",
              "text": ":x: App sync FAILED! Check ArgoCD immediately.", "icon_emoji": ":fire:"}
            - "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
      restartPolicy: Never
  backoffLimit: 2
```

---

## 5. Hook Delete Policies

By default, hook resources remain in the cluster after execution. Use delete policies to clean them up automatically:

```yaml
annotations:
  argocd.argoproj.io/hook: PostSync
  argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

| Policy | Behaviour |
|---|---|
| `HookSucceeded` | Deletes the hook resource after it completes **successfully** |
| `HookFailed` | Deletes the hook resource after it **fails** |
| `BeforeHookCreation` | Deletes any existing hook resource **before** creating the new one (default for named resources) |

> **Tip:** Combine `HookSucceeded` and `HookFailed` to always clean up:
> ```yaml
> argocd.argoproj.io/hook-delete-policy: HookSucceeded,HookFailed
> ```

### Alternative: TTL-based Cleanup

Jobs and Argo Workflows natively support `ttlSecondsAfterFinished` — no delete policy annotation needed:

```yaml
spec:
  ttlSecondsAfterFinished: 300   # auto-delete 5 minutes after completion
  template:
    ...
```

---

## 6. Hook Execution Flow

```
Sync triggered
      │
      ▼
┌─────────────────────────────────────┐
│  1. PreSync hooks run               │
│     ArgoCD waits for all to succeed │
└─────────────────┬───────────────────┘
                  │ ✓ PreSync healthy
                  ▼
┌─────────────────────────────────────┐
│  2. Sync phase                      │
│     Manifests applied + Sync hooks  │
│     ArgoCD waits for healthy state  │
└─────────────────┬───────────────────┘
                  │ ✓ All resources healthy
                  ▼
┌─────────────────────────────────────┐
│  3. PostSync hooks run              │
│     Tests, notifications, cleanup   │
└─────────────────┬───────────────────┘
                  │ ✗ Any phase fails
                  ▼
┌─────────────────────────────────────┐
│  SyncFail hooks run (on failure)    │
│  Alerts, rollback logic             │
└─────────────────────────────────────┘
```

---

## 7. Repository Structure for Lab 12

```
Lab12 hooks/
├── presync-migrate.yaml        # PreSync: DB migration Job
├── postsync-smoke-test.yaml    # PostSync: health check Job
├── postsync-notify-success.yaml # PostSync: Slack success alert
├── syncfail-notify.yaml        # SyncFail: Slack failure alert
└── app.yaml                    # ArgoCD Application manifest
```

### ArgoCD Application Manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hooks-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/amitopenwriteup/argocd-docs-labs.git
    targetRevision: HEAD
    path: Lab12 hooks
  destination:
    server: https://kubernetes.default.svc
    namespace: hooks-demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 8. `generateName` vs `name`

| Field | Behaviour | Use With |
|---|---|---|
| `generateName: schema-migrate-` | Creates unique name each sync: `schema-migrate-x7k2p` | `HookSucceeded` delete policy |
| `name: schema-migrate` | Fixed name — conflicts on second sync unless cleaned up | `BeforeHookCreation` delete policy |

> Use `generateName` when you want a fresh Job every sync. Use `name` + `BeforeHookCreation` when you want to inspect the last run's logs.

---

## 9. Applying and Verifying

### Apply the Application

```bash
kubectl apply -f app.yaml -n argocd
```

### Trigger a Sync

```bash
argocd app sync hooks-demo
```

### Watch Hook Execution

```bash
# Watch all Jobs in the namespace
kubectl get jobs -n hooks-demo -w

# Check PreSync Job logs
kubectl logs -l argocd.argoproj.io/hook=PreSync -n hooks-demo

# Check PostSync Job logs
kubectl logs -l job-name=<job-name> -n hooks-demo

# View sync status and hook phases in ArgoCD
argocd app get hooks-demo
```

### Check Sync History

```bash
argocd app history hooks-demo
```

---

## 10. Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| Hook Job stuck in `Pending` | Insufficient cluster resources | Check `kubectl describe pod` for scheduling errors |
| PreSync never completes | Job fails and `backoffLimit` exceeded | Check logs with `kubectl logs <pod>` |
| PostSync not running | Sync phase resources not Healthy | Inspect app health: `argocd app get <name>` |
| Hook runs every sync | Missing delete policy | Add `argocd.argoproj.io/hook-delete-policy` |
| Old hook Job conflicts | Using `name` without `BeforeHookCreation` | Switch to `generateName` or add `BeforeHookCreation` |
| `SyncFail` hook not triggered | Sync succeeded (no failure) | Intentionally break a manifest to test |

---

## 11. Hook Annotations Quick Reference

```yaml
metadata:
  annotations:
    # Hook phase
    argocd.argoproj.io/hook: PreSync

    # Delete policy
    argocd.argoproj.io/hook-delete-policy: HookSucceeded

    # Sync wave within a hook phase (optional ordering)
    argocd.argoproj.io/sync-wave: "1"
```

### All Valid Hook Values

```
PreSync | Sync | PostSync | SyncFail | Skip
```

### All Valid Delete Policy Values

```
HookSucceeded | HookFailed | BeforeHookCreation
```

---

*Resource hooks enable reliable, ordered deployment workflows — ensuring databases are migrated before pods start, health is verified before traffic is routed, and teams are notified the moment something goes wrong.*
