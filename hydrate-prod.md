# Argo CD Source Hydrator Lab (prod)

Repo: `https://github.com/amitopenwriteup/kustomize-argo.git`

```
kustomize-argo/
└── overlays/
    └── prod/
```

Flow: hydrator renders `overlays/prod` on `main` → pushes to `environments/prod-next` → you merge a PR into `environments/prod` → Argo CD syncs.

---

## 1. Install

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install-with-hydrator.yaml
```

## 2. Enable hydrator

```bash
kubectl patch configmap argocd-cmd-params-cm -n argocd \
  --type merge -p '{"data":{"hydrator.enabled":"true"}}'

kubectl rollout restart statefulset argocd-application-controller -n argocd
kubectl rollout restart deployment argocd-repo-server argocd-server -n argocd
```

## 3. Git write credentials

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

Also add the repo for read access (`argocd repo add` or a `repository` secret).

---

## 4. Application

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
      path: apps/my-app/overlays/prod
    hydrateTo:
      targetBranch: environments/prod-next
```

Apply:

```bash
kubectl apply -f myapp-prod.yaml
```

---

## 5. What exists vs. what is created

| Item | Value | Must exist beforehand? |
|---|---|---|
| Dry source path (on `main`) | `overlays/prod` | **Yes** |
| Sync branch | `environments/prod` | No, created by Argo CD (initial commit) |
| Hydrate-to branch | `environments/prod-next` | No, created by Argo CD |
| Output path | `apps/my-app/overlays/prod` | No, do not create manually |

Until the PR is merged, the app shows `app path does not exist - waiting for an external process...`. This is expected.

---

## 6. Pull request and merge (GitHub)

1. Open the repo on GitHub → **Pull requests** → **New pull request**.
2. Set the branches:
   - **base:** `environments/prod`
   - **compare:** `environments/prod-next`

   (The default base is `main`. Change it.)
3. Check the diff: `manifest.yaml`, `hydrator.metadata`, `.gitattributes`.
4. **Create pull request** → **Merge pull request** → **Confirm merge**.
5. In Argo CD, open the app → **Refresh → Hard Refresh**.

CLI alternative:

```bash
gh pr create --repo amitopenwriteup/kustomize-argo \
  --base environments/prod --head environments/prod-next \
  --title "Promote prod" --body "Hydrated manifests"
gh pr merge --repo amitopenwriteup/kustomize-argo --merge
```

Never merge into `main`. Do not delete `environments/prod-next`.

---

## 7. Verify

```bash
argocd app get myapp-prod
kubectl get all -n myapp-prod
```

Expected: app **Synced** and **Healthy**.

---

## 8. Troubleshooting

| Symptom | Fix |
|---|---|
| `app path does not exist - waiting for an external process` | Merge `environments/prod-next` into `environments/prod`, then Hard Refresh |
| Hydrate To set to `main` | Use `environments/prod-next` |
| Push or permission error | Check the `repository-write` secret and branch protection |
| Nothing hydrated | Confirm `hydrator.enabled: "true"`, restart components, check `argocd-commit-server` logs |

To skip the PR step, remove the `hydrateTo` block and the hydrator writes straight to `environments/prod`.
