# ArgoCD CLI — Install & Login
**Kind Cluster | kubectl | GitHub | GitOps**  
*End-to-End Setup & Application Deployment*  
*February 2026*

---

## Install the ArgoCD CLI

Download and install the latest ArgoCD CLI binary:

```bash
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

rm argocd-linux-amd64
```

---

## Log In via CLI

**1.** Get your machine's IP address:

```bash
hostname -i
```

**2.** Log in to the ArgoCD server using the IP and port:

```bash
argocd login <your-ip>:8080
```

**Example:**

```bash
argocd login 172.17.7.152:8080
```

When prompted, enter your credentials:

| Field | Value |
|-------|-------|
| **Username** | `admin` |
| **Password** | Your ArgoCD admin password |

> ⚠️ **Note:** If you are using a self-signed certificate (default for local Kind clusters), you may be prompted to confirm the insecure connection. You can bypass certificate verification by appending `--insecure` to the login command:
> ```bash
> argocd login 172.17.7.152:8080 --insecure
> ```

A successful login will display:
```
'admin:login' logged in successfully
```
