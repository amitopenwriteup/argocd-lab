# ArgoCD RBAC — Simple Local Users Example

> **Goal:** Set up ArgoCD RBAC using only local users and roles — no SSO, no OIDC.
> **Time:** ~15 minutes.
> **Use case:** Lab environments, air-gapped clusters, quick demos, initial bootstrap.

---

## What We're Building

```
3 local users → 3 roles → scoped permissions

alice  (admin)     → full access
bob    (deployer)  → sync staging only
carol  (viewer)    → read-only everywhere
```

---

## Step 1 — Define Local Users in argocd-cm

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # Declare local users (comma-separated capabilities: login, apiKey)
  accounts.alice: login
  accounts.bob:   login
  accounts.carol: login
```

```bash
kubectl apply -f argocd-cm.yaml
```

---

## Step 2 — Set Passwords via CLI

```bash
# Log in as the default admin first
argocd login <ARGOCD_SERVER> --username admin --password <ADMIN_PASSWORD> --insecure

# Set passwords for each user
argocd account update-password --account alice --new-password Alice@1234!
argocd account update-password --account bob   --new-password Bob@1234!
argocd account update-password --account carol --new-password Carol@1234!
```

---

## Step 3 — Define Roles and Bind Users in argocd-rbac-cm

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly   # fallback for any user with no explicit binding
  policy.matchMode: glob

  policy.csv: |
    # --- role: deployer ---
    # Can view everything, sync staging only, no deletes, no exec
    p, role:deployer, applications, get,      */*, allow
    p, role:deployer, applications, sync,     staging/*, allow
    p, role:deployer, applications, override, staging/*, allow
    p, role:deployer, logs,         get,      */*, allow
    p, role:deployer, exec,         *,        */*, deny

    # --- User → role bindings ---
    g, alice, role:admin      # built-in admin role — full access
    g, bob,   role:deployer   # custom role defined above
    g, carol, role:readonly   # built-in read-only role
```

```bash
kubectl apply -f argocd-rbac-cm.yaml
```

---

## Step 4 — Verify

```bash
# Test as alice (admin — should be allowed everything)
argocd login <ARGOCD_SERVER> --username alice --password Alice@1234! --insecure
argocd app list
argocd app sync <any-app>

# Test as bob (deployer — sync staging allowed, production denied)
argocd login <ARGOCD_SERVER> --username bob --password Bob@1234! --insecure
argocd app sync staging-app      # ✅ allowed
argocd app sync production-app   # ❌ PermissionDenied

# Test as carol (viewer — no mutations)
argocd login <ARGOCD_SERVER> --username carol --password Carol@1234! --insecure
argocd app list                  # ✅ allowed
argocd app sync staging-app      # ❌ PermissionDenied
```

**Verify permissions without logging in as the user (as admin):**
```bash
argocd admin settings rbac can role:deployer sync  applications 'staging/my-app'   # yes
argocd admin settings rbac can role:deployer sync  applications 'production/my-app' # no
argocd admin settings rbac can role:readonly  get   applications '*//*'              # yes
argocd admin settings rbac can role:readonly  delete applications '*//*'             # no
```

---

## Summary

| User | Role | Can View | Can Sync Staging | Can Sync Production | Can Delete | Can Exec |
|------|------|----------|-----------------|---------------------|------------|----------|
| alice | `role:admin` | ✅ | ✅ | ✅ | ✅ | ✅ |
| bob | `role:deployer` | ✅ | ✅ | ❌ | ❌ | ❌ |
| carol | `role:readonly` | ✅ | ❌ | ❌ | ❌ | ❌ |
| *(anyone else)* | `policy.default` | ✅ | ❌ | ❌ | ❌ | ❌ |

---

## Quick Troubleshooting

| Problem | Fix |
|---------|-----|
| User can't log in | Check `accounts.<name>: login` is in `argocd-cm` |
| User has no permissions after login | Check `g, <username>, role:<name>` exists in `policy.csv` |
| Sync denied unexpectedly | Verify `policy.matchMode: glob` is set; check object pattern |
| `role:admin` not working | `role:admin` is a built-in global role — no need to define policies for it |

---

*Three users, two ConfigMaps, fifteen minutes. Add SSO groups later when you're ready to scale.*
