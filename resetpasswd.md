# ArgoCD — Reset Admin Password
**Kind Cluster | kubectl | GitHub | GitOps**  
*End-to-End Setup & Application Deployment*  
*February 2026*

---

## Overview

The password stored in the `argocd-secret` Kubernetes secret is double-encoded:

1. **Bcrypt hash** — the actual password hash
2. **Base64 encoded** — Kubernetes secret requirement

The steps below use `htpasswd` to generate a new bcrypt hash and patch the ArgoCD secret directly.

---

## Step 1 — Install htpasswd

```bash
apt install apache2-utils
```

---

## Step 2 — Generate a New Bcrypt Hash

Set your desired password and generate the bcrypt hash:

```bash
NEW_PASSWORD="newpassword"

BCRYPT_HASH=$(htpasswd -nbBC 10 "" $NEW_PASSWORD | tr -d ':\n' | sed 's/$2y/$2a/')
```

### Command Breakdown

| Part | Description |
|------|-------------|
| `htpasswd -nbBC 10` | Generate a bcrypt hash with cost factor 10, no file output |
| `tr -d ':\n'` | Strip the leading colon and newline from the output |
| `sed 's/$2y/$2a/'` | Replace the `$2y` bcrypt prefix with `$2a` for ArgoCD compatibility |

---

## Step 3 — Patch the ArgoCD Secret

Apply the new hash and update the password modification timestamp:

```bash
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "admin.password": "'$BCRYPT_HASH'",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'
```

> ℹ️ The `admin.passwordMtime` field records when the password was last changed. ArgoCD uses this to invalidate existing sessions after a password reset.

---

## Step 4 — Verify the Change

Log in via the CLI to confirm the new password is active:

```bash
argocd login <your-ip>:8080 --username admin --password newpassword --insecure
```

> ⚠️ **Security Note:** Replace `newpassword` with a strong password before use in any shared or production environment.
