# Argo CD Source Hydrator Setup (Kustomize overlays)

Repo: `https://github.com/amitopenwriteup/kustomize-argo.git`

```
kustomize-argo/
└── overlays/
    ├── dev/
    └── prod/
```

**How it works:** the hydrator runs `kustomize build` on the *dry source* (`overlays/dev` on `main`), commits the rendered `manifest.yaml` to a Git branch, and Argo CD syncs from that branch.

---

## 1. Install Argo CD with the commit server

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install-with-hydrator.yaml
```

For a core/headless install use `core-install-with-hydrator.yaml`. The hydrator needs the extra `argocd-commit-server` component. Check the docs for your Argo CD version, as the feature is still evolving.

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

## 4. Sample Application (with PR promotion)

```yaml
project: default
destination:
  server: https://kubernetes.default.svc
  namespace: myapp
syncPolicy:
  automated:
    prune: false
    selfHeal: false
    enabled: true
  syncOptions:
    - CreateNamespace=true
sourceHydrator:
  drySource:
    repoURL: https://github.com/amitopenwriteup/kustomize-argo.git
    targetRevision: HEAD
    path: overlays/dev
  syncSource:
    targetBranch: environments/dev
    path: apps/my-app/overlays/dev
  hydrateTo:
    targetBranch: environments/dev-next
```

### What exists, and what gets created

| Item | Value | Must exist beforehand? | Who creates it |
|---|---|---|---|
| Dry source branch | `HEAD` (e.g. `main`) | **Yes** | You |
| Dry source path | `overlays/dev` | **Yes** | You (your existing Kustomize overlay) |
| Hydrate-to branch | `environments/dev-next` | **No, must not be pre-created by hand** | Hydrator, on first run |
| Hydrated output path | `apps/my-app/overlays/dev` (inside `dev-next`) | **No, should not exist** | Hydrator, on first run |
| Sync branch | `environments/dev` | **No** | Created when you merge the PR from `dev-next` |
| Sync path | `apps/my-app/overlays/dev` (inside `environments/dev`) | **No, should not exist** | Appears after the PR merge |

Key points:

- Only the **dry source** (`overlays/dev`) must exist. Everything under `environments/*` is output, so do not create those branches or folders manually.
- `apps/my-app/overlays/dev` is **only a name for the output folder**. It does not need to match your repo layout. `overlays/dev` would work equally well.
- The hydrator **owns** that output folder and overwrites its contents each time. Do not edit it by hand, and do not let two apps share the same branch + path.

### What happens step by step

1. Hydrator reads `overlays/dev` on `main` and runs `kustomize build`.
2. Because `hydrateTo` is set, it creates branch **`environments/dev-next`** and writes:
   ```
   apps/my-app/overlays/dev/manifest.yaml
   apps/my-app/overlays/dev/hydrator.metadata
   ```
3. Argo CD syncs from `environments/dev`, which does not have that path yet, so the app shows:
   > `app path does not exist - waiting for an external process to update environments/dev from environments/dev-next`

   **This is expected, not a failure.**
4. You open a PR in GitHub from `environments/dev-next` into `environments/dev` and merge it.
5. `environments/dev` now contains `apps/my-app/overlays/dev/manifest.yaml`. Argo CD syncs it. Repeat the PR for every future change.

---

## 5. Without PR promotion (automatic)

Remove the `hydrateTo` block and the hydrator writes straight to `environments/dev`:

```yaml
sourceHydrator:
  drySource:
    repoURL: https://github.com/amitopenwriteup/kustomize-argo.git
    targetRevision: HEAD
    path: overlays/dev
  syncSource:
    targetBranch: environments/dev
    path: apps/my-app/overlays/dev
```

Then `environments/dev` is created by the hydrator on the first run, and the app syncs without a PR. For prod, use `overlays/prod` as the dry path and `environments/prod` as the branch.

---

## 6. Rules

- Use **one branch per environment** (`environments/dev`, `environments/prod`).
- Never use `main` as `syncSource.targetBranch` or `hydrateTo.targetBranch`. It is your dry source.
- Keep `syncSource.path` consistent between runs. Changing it creates a new folder.


---

