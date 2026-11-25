# Argo CD Source Hydrator Setup (Kustomize overlays)

Repo: `https://github.com/amitopenwriteup/kustomize-argo.git`

```
kustomize-argo/
└── overlays/
    ├── dev/
    └── prod/
```

**How it works:** the hydrator runs `kustomize build` on the *dry source* (`overlays/dev` on `main`), commits the rendered `manifest.yaml` to an *environment branch* (`environments/dev`), and Argo CD syncs from that branch.

---

## 1. Install Argo CD with the commit server

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install-with-hydrator.yaml
```

For a core/headless install use `core-install-with-hydrator.yaml`.
The hydrator needs the extra `argocd-commit-server` component. Check the docs for your Argo CD version, as the feature is still evolving.

## 2. Enable the hydrator

```bash
kubectl patch configmap argocd-cmd-params-cm -n argocd \
  --type merge -p '{"data":{"hydrator.enabled":"true"}}'

kubectl rollout restart statefulset argocd-application-controller -n argocd
kubectl rollout restart deployment argocd-repo-server argocd-server -n argocd
```

## 3. Git credentials

The hydrator **pushes commits**, so it needs **write** credentials in addition to normal read access.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: repo-write-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository-write
stringData:
  type: git
  url: https://github.com/amitopenwriteup/kustomize-argo.git
  username: <github-user>
  password: <token-with-write-access>
```

Also register the repo for reading (`argocd repo add` or a normal `repository` secret).

---

## 4. Application manifests

### Dev

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
    syncOptions:
      - CreateNamespace=true
  sourceHydrator:
    drySource:
      repoURL: https://github.com/amitopenwriteup/kustomize-argo.git
      targetRevision: HEAD
      path: overlays/dev
    syncSource:
      targetBranch: environments/dev
      path: overlays/dev
```

### Prod

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-prod
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
    syncOptions:
      - CreateNamespace=true
  sourceHydrator:
    drySource:
      repoURL: https://github.com/amitopenwriteup/kustomize-argo.git
      targetRevision: HEAD
      path: overlays/prod
    syncSource:
      targetBranch: environments/prod
      path: overlays/prod
```

---

## 5. Field reference

| Field | Meaning | Must already exist? |
|---|---|---|
| `drySource.path` | Your existing Kustomize overlay folder | **Yes**, on `targetRevision` |
| `syncSource.targetBranch` | Branch the hydrator writes rendered manifests to | No, created on first run |
| `syncSource.path` | Folder inside that branch for `manifest.yaml` | No, created on first run |
| `hydrateTo.targetBranch` | Optional "next" branch for PR-based promotion | No |

Rules:

- Use **one branch per environment** (`environments/dev`, `environments/prod`).
- Never use `main` as `targetBranch` or `hydrateTo`. It is your dry source.
- Each app owns its `targetBranch` + `path`. Two apps must not share the same combination, because the hydrator overwrites that folder.
- Mirroring the dry path (`overlays/dev`) in `syncSource.path` is the clearest convention.
- Do not edit the hydrated branches by hand.

---

## 6. Optional: PR-based promotion

To review rendered manifests before they go live, add `hydrateTo`:

```yaml
  sourceHydrator:
    # drySource / syncSource as above
    hydrateTo:
      targetBranch: environments/dev-next
```

Flow:

1. Hydrator pushes to `environments/dev-next`.
2. You open a PR from `environments/dev-next` into `environments/dev`.
3. After merge, Argo CD syncs from `environments/dev`.

Until the PR is merged, the app shows the error described below. That is expected.

---

## 7. Verify

```bash
argocd app get myapp-dev
kubectl logs -n argocd deploy/argocd-commit-server
```

GitHub should show branch `environments/dev` containing:

```
overlays/dev/manifest.yaml
overlays/dev/hydrator.metadata
```

---

## 8. Troubleshooting

### `app path does not exist - waiting for an external process to update environments/dev from environments/dev-next`

**Cause:** `hydrateTo` is set, so output goes to `environments/dev-next` and waits for a PR merge into `environments/dev`.

**Fix:** either remove the `hydrateTo` block (automatic mode), or merge the PR from `environments/dev-next` into `environments/dev`. Then do a **Hard Refresh** on the app.

### Hydrate To is set to `main`

**Cause:** `hydrateTo` must be a separate branch. Pointing it at your dry source branch makes the hydrator try to commit rendered files into `main`.

**Fix:** clear it, or use a dedicated branch such as `environments/dev-next`.

### Push or permission errors

- Confirm the `repository-write` secret exists and the token has write access.
- If branch protection is on, allow the token to create and push `environments/*` branches, or create the branches manually.

### Nothing is hydrated

- Confirm `hydrator.enabled: "true"` and that the components were restarted.
- Confirm `argocd-commit-server` is running.
- Check the commit-server and application-controller logs.

---

## Quick checklist

- [ ] Installed with the hydrator manifest (commit-server running)
- [ ] `hydrator.enabled: "true"` and components restarted
- [ ] `repository-write` secret created
- [ ] `drySource.path` matches an existing overlay
- [ ] `syncSource.targetBranch` is unique per environment and is not `main`
- [ ] `hydrateTo` removed (automatic) or PR flow understood
- [ ] Hard Refresh done after changes
