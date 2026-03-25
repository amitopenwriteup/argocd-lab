# ArgoCD RBAC Workshop
**Hands-On Technical Workshop | Role-Based Access Control in ArgoCD**

> **Audience:** Platform Engineers, DevOps Engineers, Security Engineers, Team Leads
> **Prerequisites:** Basic familiarity with Kubernetes, GitOps concepts, and ArgoCD UI
> **Format:** Concept → Demo → Lab exercise per section
> **Total Duration:** ~4 Hours

---

## Workshop at a Glance

| # | Section | Theme | Duration |
|---|---------|--------|----------|
| 1 | RBAC Foundations | *Why and what* | 30 min |
| 2 | ArgoCD's Permission Model | *How ArgoCD thinks about access* | 30 min |
| 3 | Configuring RBAC — Core Syntax | *Writing your first policies* | 45 min |
| — | **Break** | | 10 min |
| 4 | SSO & Group Integration | *Connecting your identity provider* | 45 min |
| 5 | Advanced RBAC Patterns | *Multi-tenant, project-scoped, least privilege* | 45 min |
| — | **Break** | | 10 min |
| 6 | Labs & Exercises | *Hands-on policy authoring* | 45 min |
| 7 | Troubleshooting & Best Practices | *When things go wrong* | 20 min |
| — | **Wrap-up & Q&A** | | 10 min |

---

## Section 1 — RBAC Foundations
**Duration: 30 minutes**

### 1.1 Why RBAC in ArgoCD Matters

ArgoCD is a **privileged system** — it has write access to your Kubernetes clusters. Without proper RBAC:
- Any authenticated user can sync, delete, or override any application
- There is no separation between prod and non-prod access
- Audit trails are meaningless if everyone is `admin`

> **Key question for your organisation:** *Who should be able to press "Sync" against production?*

### 1.2 The Principle of Least Privilege — Applied to GitOps

| Access Level | Who Needs It | Example |
|-------------|-------------|---------|
| Read-only | Developers, stakeholders, on-call engineers | View app health and logs |
| Deployer | Senior devs, team leads | Sync to staging |
| Operator | Platform engineers | Sync to production, manage resources |
| Admin | ArgoCD platform team only | Manage repositories, clusters, RBAC itself |

### 1.3 ArgoCD's Two Access Boundaries

1. **ArgoCD RBAC** — controls what users can do *within* ArgoCD (sync, delete, create apps, manage projects)
2. **Kubernetes RBAC** — controls what ArgoCD's *service account* can do on the cluster

> These are **independent systems**. A user with ArgoCD admin rights cannot do anything in Kubernetes directly — ArgoCD acts on their behalf using its own service account. You must secure both layers.

---

## Section 2 — ArgoCD's Permission Model
**Duration: 30 minutes**

### 2.1 Core Concepts

#### Projects
ArgoCD **Projects** (`AppProject` CRD) are the primary boundary for multi-tenancy. They define:
- Which **source repos** are allowed
- Which **destination clusters and namespaces** are allowed
- Which **resource kinds** can be deployed
- Which **roles** exist and what they can do

```yaml
# Example AppProject
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-payments
  namespace: argocd
spec:
  description: Payments team project
  sourceRepos:
    - https://gitlab.example.com/payments/*
  destinations:
    - namespace: payments-*
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  roles:
    - name: deployer
      description: Can sync but not delete
      policies:
        - p, proj:team-payments:deployer, applications, sync, team-payments/*, allow
      groups:
        - payments-engineers
```

#### Applications
An **Application** (`Application` CRD) is a deployed workload managed by ArgoCD. RBAC permissions are applied at the application level within a project.

#### The `default` Project
All applications belong to a project. The `default` project has no restrictions — **do not use it in production**. Migrate everything to named projects.

### 2.2 Built-in Roles

| Role | Capabilities |
|------|-------------|
| `role:readonly` | View all apps, projects, repos, clusters — no mutations |
| `role:admin` | Full access to everything — equivalent to cluster admin in ArgoCD |

> **Warning:** `role:admin` is a global role. It bypasses all project-level restrictions.

### 2.3 Permission Subjects

ArgoCD RBAC can grant permissions to:
- **Local users** — defined in `argocd-cm` ConfigMap
- **SSO users** — authenticated via Dex or an external OIDC provider
- **SSO groups** — the recommended approach for teams

### 2.4 Where RBAC Config Lives

```
argocd namespace
├── argocd-cm (ConfigMap)        → local users, SSO connector config, policy.default
├── argocd-rbac-cm (ConfigMap)   → policy.csv, policy rules
└── AppProject (CRD)             → project-scoped roles and group bindings
```

---

## Section 3 — Configuring RBAC — Core Syntax
**Duration: 45 minutes**

### 3.1 The Policy Language

ArgoCD uses **Casbin** policy syntax. Every rule is a `p` (policy) statement or a `g` (group) statement.

#### Policy statement syntax
```
p, <subject>, <resource>, <action>, <object>, <effect>
```

| Field | Description | Examples |
|-------|-------------|---------|
| `subject` | Who | `role:dev`, `user:alice`, `proj:myproject:deployer` |
| `resource` | What type | `applications`, `clusters`, `repositories`, `projects`, `logs`, `exec` |
| `action` | What operation | `get`, `create`, `update`, `delete`, `sync`, `override`, `action` |
| `object` | Specific target | `*/*, myproject/myapp, default/*` |
| `effect` | Allow or deny | `allow`, `deny` |

#### Group assignment syntax
```
g, <user-or-sso-group>, <role>
```

### 3.2 Resource & Action Reference

```
Resources:          Actions:
applications        get
clusters            create
repositories        update
projects            delete
accounts            sync
certificates        override
gpgkeys             action
logs                *  (wildcard — all actions)
exec
```

### 3.3 Writing Your First Policies

#### Read-only access to all applications
```csv
# argocd-rbac-cm → policy.csv
p, role:viewer, applications, get, */*, allow
p, role:viewer, projects,     get, *,    allow

g, oidc:developers, role:viewer
```

#### Deploy to staging only
```csv
p, role:staging-deployer, applications, get,    */*, allow
p, role:staging-deployer, applications, sync,   staging/*, allow
p, role:staging-deployer, applications, override, staging/*, allow
p, role:staging-deployer, logs,         get,    staging/*, allow

g, oidc:team-alpha, role:staging-deployer
```

#### Production operator — named project only
```csv
p, role:prod-operator, applications, get,    production/*, allow
p, role:prod-operator, applications, sync,   production/*, allow
p, role:prod-operator, applications, delete, production/*, deny
p, role:prod-operator, clusters,     get,    *,            allow

g, oidc:platform-engineers, role:prod-operator
```

### 3.4 The `argocd-rbac-cm` ConfigMap in Full

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  # Default role for authenticated users who have no other role assigned
  policy.default: role:readonly

  # CSV policy block
  policy.csv: |
    # --- Viewers ---
    p, role:viewer, applications, get, */*, allow
    p, role:viewer, projects,     get, *,   allow

    # --- Staging deployers ---
    p, role:staging-deployer, applications, get,      staging/*, allow
    p, role:staging-deployer, applications, sync,     staging/*, allow
    p, role:staging-deployer, applications, override, staging/*, allow

    # --- Production operators ---
    p, role:prod-operator, applications, get,  production/*, allow
    p, role:prod-operator, applications, sync, production/*, allow

    # --- Group bindings ---
    g, oidc:developers,        role:viewer
    g, oidc:team-leads,        role:staging-deployer
    g, oidc:platform-engineers, role:prod-operator

  # Enable RBAC enforcement (set to false only during initial setup)
  policy.matchMode: glob
```

### 3.5 Object Patterns — Glob Matching

```
*//*            → all projects, all applications
production//*   → all apps in the production project
staging/frontend → only the frontend app in staging project
*//*-service    → all apps ending in "-service" across all projects
```

> **Note:** `policy.matchMode: glob` must be set for wildcard patterns. Default is exact match.

---

## ☕ Break — 10 Minutes

---

## Section 4 — SSO & Group Integration
**Duration: 45 minutes**

### 4.1 SSO Architecture in ArgoCD

```
User Browser
    │
    ▼
ArgoCD UI / API
    │ redirects to
    ▼
Dex (built-in OIDC proxy)
    │ connects to
    ▼
Identity Provider (GitLab, GitHub, Okta, Azure AD, LDAP...)
    │ returns groups + claims
    ▼
ArgoCD RBAC policy.csv evaluates group membership
```

ArgoCD ships with **Dex** as an embedded OIDC provider that federates to your IdP. You can also point ArgoCD directly at an external OIDC provider (skip Dex).

### 4.2 Configuring Dex — GitLab Example

```yaml
# argocd-cm ConfigMap
data:
  url: https://argocd.example.com

  dex.config: |
    connectors:
      - type: gitlab
        id: gitlab
        name: GitLab
        config:
          baseURL: https://gitlab.example.com
          clientID: $dex.gitlab.clientID
          clientSecret: $dex.gitlab.clientSecret
          redirectURI: https://argocd.example.com/api/dex/callback
          groups:
            - platform-engineers
            - team-alpha
            - team-beta
            - readonly-users
          useLoginAsID: false
```

> **Secret management:** Store `clientID` and `clientSecret` in a Kubernetes Secret, then reference with `$secret-key` syntax in the ConfigMap.

### 4.3 Configuring Dex — GitHub Example

```yaml
dex.config: |
  connectors:
    - type: github
      id: github
      name: GitHub
      config:
        clientID: $dex.github.clientID
        clientSecret: $dex.github.clientSecret
        redirectURI: https://argocd.example.com/api/dex/callback
        orgs:
          - name: my-github-org
            teams:
              - platform
              - developers
              - readonly
```

### 4.4 Configuring Dex — Okta / Generic OIDC

```yaml
dex.config: |
  connectors:
    - type: oidc
      id: okta
      name: Okta
      config:
        issuer: https://yourcompany.okta.com
        clientID: $dex.okta.clientID
        clientSecret: $dex.okta.clientSecret
        redirectURI: https://argocd.example.com/api/dex/callback
        scopes:
          - openid
          - profile
          - email
          - groups
        claimMapping:
          groups: groups   # OIDC claim name that contains group membership
        insecureEnableGroups: true
```

### 4.5 Mapping SSO Groups to ArgoCD Roles

Once Dex returns group claims, bind them in `argocd-rbac-cm`:

```csv
# Format: g, <oidc-prefix>:<group-name>, <role>

g, oidc:platform-engineers,  role:admin
g, oidc:team-alpha,           role:staging-deployer
g, oidc:team-beta,            role:staging-deployer
g, oidc:readonly-users,       role:readonly

# GitHub org/team syntax
g, my-github-org:platform,   role:admin
g, my-github-org:developers, role:viewer
```

> **Prefix matters:** Use `oidc:` for Dex-federated groups, the GitHub org name for GitHub teams, and check your IdP connector docs for the exact claim format.

### 4.6 Local Users (Break-Glass Accounts)

Define local users in `argocd-cm`:

```yaml
data:
  accounts.breakglass: apiKey, login
  accounts.breakglass.enabled: "true"
```

Then assign a role:
```csv
g, breakglass, role:admin
```

Set the password:
```bash
argocd account update-password \
  --account breakglass \
  --new-password <strong-password> \
  --current-password <admin-password>
```

> Store break-glass credentials in your secrets manager (Vault, AWS Secrets Manager). Rotate after each use. Alert on login.

---

## Section 5 — Advanced RBAC Patterns
**Duration: 45 minutes**

### 5.1 Multi-Tenant Architecture

The recommended structure for organisations with multiple teams:

```
ArgoCD Instance
├── AppProject: team-payments
│   ├── Allowed repos: gitlab.example.com/payments/*
│   ├── Allowed destinations: payments-* namespaces
│   └── Roles: payments-deployer, payments-viewer
├── AppProject: team-identity
│   ├── Allowed repos: gitlab.example.com/identity/*
│   ├── Allowed destinations: identity-* namespaces
│   └── Roles: identity-deployer
└── AppProject: platform
    ├── Allowed repos: gitlab.example.com/platform/*
    ├── Allowed destinations: * (all namespaces)
    └── Roles: platform-operator
```

**AppProject for team-payments:**
```yaml
spec:
  roles:
    - name: deployer
      description: Sync apps in payments project
      policies:
        - p, proj:team-payments:deployer, applications, get,    team-payments/*, allow
        - p, proj:team-payments:deployer, applications, sync,   team-payments/*, allow
        - p, proj:team-payments:deployer, applications, override, team-payments/*, allow
      groups:
        - oidc:payments-engineers
    - name: viewer
      policies:
        - p, proj:team-payments:viewer, applications, get, team-payments/*, allow
      groups:
        - oidc:payments-stakeholders
```

### 5.2 Environment Separation Pattern

Separate AppProjects per environment, with escalating access requirements:

```csv
# Developers: read everywhere, deploy only to dev
p, role:developer, applications, get,  */*, allow
p, role:developer, applications, sync, dev/*, allow

# Team leads: read everywhere, deploy to dev and staging
p, role:team-lead, applications, get,  */*, allow
p, role:team-lead, applications, sync, dev/*, allow
p, role:team-lead, applications, sync, staging/*, allow

# Platform engineers: full deployment rights
p, role:platform-eng, applications, get,    */*, allow
p, role:platform-eng, applications, sync,   */*, allow
p, role:platform-eng, applications, delete, */*, allow
p, role:platform-eng, clusters,     get,    *,   allow
p, role:platform-eng, repositories, get,    *,   allow
```

### 5.3 Restricting Dangerous Actions

Explicitly deny high-risk operations, even for elevated roles:

```csv
# Allow syncs but prevent resource deletion
p, role:deployer, applications, sync,   production/*, allow
p, role:deployer, applications, delete, production/*, deny

# Allow log access but block exec (terminal access into pods)
p, role:developer, logs, get,  */*, allow
p, role:developer, exec, *,    */*, deny

# Explicit denies take precedence over allows in ArgoCD
```

### 5.4 Sync Windows — Time-Based Access Control

Complement RBAC with sync windows to enforce change freeze periods:

```yaml
# In AppProject spec
spec:
  syncWindows:
    # Only allow syncs Mon-Fri 09:00-17:00 UTC for production
    - kind: allow
      schedule: "0 9 * * 1-5"
      duration: 8h
      applications:
        - "*"
      namespaces:
        - production-*
      manualSync: false   # set true to allow manual override

    # Deny all syncs during release freeze
    - kind: deny
      schedule: "0 0 * * 5"   # Every Friday midnight
      duration: 48h
      applications:
        - "*"
```

### 5.5 Resource Allow/Deny Lists in AppProjects

Prevent teams from deploying cluster-level resources they shouldn't touch:

```yaml
spec:
  # Whitelist: only these cluster-scoped resources allowed
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
    - group: 'networking.k8s.io'
      kind: IngressClass

  # Blacklist: these namespace-scoped resources are forbidden
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
    - group: ''
      kind: LimitRange

  # Whitelist (preferred): only these namespace-scoped resources allowed
  namespaceResourceWhitelist:
    - group: 'apps'
      kind: Deployment
    - group: 'apps'
      kind: StatefulSet
    - group: ''
      kind: Service
    - group: ''
      kind: ConfigMap
    - group: ''
      kind: Secret
```

---

## ☕ Break — 10 Minutes

---

## Section 6 — Labs & Exercises
**Duration: 45 minutes**

### Lab Prerequisites

```bash
# Verify ArgoCD CLI is installed
argocd version

# Log in to the workshop ArgoCD instance
argocd login <ARGOCD_SERVER> \
  --username admin \
  --password <PASSWORD> \
  --insecure

# Verify kubectl context
kubectl config current-context
kubectl get ns argocd
```

---

### Lab 1 — Create a Local User and Assign a Role (15 min)

**Objective:** Create a `dev-user` local account with read-only access.

**Step 1: Add user to argocd-cm**
```bash
kubectl -n argocd patch configmap argocd-cm \
  --patch '{"data": {"accounts.dev-user": "apiKey, login", "accounts.dev-user.enabled": "true"}}'
```

**Step 2: Set the password**
```bash
argocd account update-password \
  --account dev-user \
  --new-password Workshop@2024! \
  --current-password <admin-password>
```

**Step 3: Add RBAC policy**
```bash
kubectl -n argocd patch configmap argocd-rbac-cm \
  --patch '{"data": {"policy.csv": "g, dev-user, role:readonly\n"}}'
```

**Step 4: Verify**
```bash
argocd login <ARGOCD_SERVER> \
  --username dev-user \
  --password Workshop@2024! \
  --insecure

# Try to list apps (should work)
argocd app list

# Try to sync an app (should fail with permission denied)
argocd app sync <any-app-name>
```

**Expected result:** `app list` succeeds, `app sync` returns `PermissionDenied`.

---

### Lab 2 — Create an AppProject with Scoped Roles (15 min)

**Objective:** Create a project for `team-workshop` with a deployer role.

**Step 1: Create the AppProject**
```yaml
# Save as team-workshop-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-workshop
  namespace: argocd
spec:
  description: Workshop team project
  sourceRepos:
    - '*'
  destinations:
    - namespace: workshop-*
      server: https://kubernetes.default.svc
  namespaceResourceWhitelist:
    - group: 'apps'
      kind: Deployment
    - group: ''
      kind: Service
    - group: ''
      kind: ConfigMap
  roles:
    - name: deployer
      description: Can sync workshop apps
      policies:
        - p, proj:team-workshop:deployer, applications, get,  team-workshop/*, allow
        - p, proj:team-workshop:deployer, applications, sync, team-workshop/*, allow
      groups:
        - workshop-participants
```

```bash
kubectl apply -f team-workshop-project.yaml
argocd proj get team-workshop
```

**Step 2: Create a test application inside the project**
```bash
argocd app create workshop-app \
  --project team-workshop \
  --repo https://github.com/argoproj/argocd-example-apps \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace workshop-test
```

**Step 3: Verify project-scoped policy**
```bash
argocd proj role list team-workshop
argocd proj role get team-workshop deployer
```

---

### Lab 3 — Test a Deny Rule (15 min)

**Objective:** Confirm explicit deny prevents a sync to production.

**Step 1: Add deny rule to argocd-rbac-cm**
```yaml
# Update argocd-rbac-cm
data:
  policy.csv: |
    p, role:dev-deployer, applications, get,    */*, allow
    p, role:dev-deployer, applications, sync,   staging/*, allow
    p, role:dev-deployer, applications, sync,   production/*, deny

    g, dev-user, role:dev-deployer
```

```bash
kubectl apply -f argocd-rbac-cm.yaml
```

**Step 2: Test as dev-user**
```bash
# Re-login as dev-user
argocd login <ARGOCD_SERVER> --username dev-user --password Workshop@2024! --insecure

# Sync to staging — should succeed if app exists
argocd app sync staging-app

# Sync to production — should be denied
argocd app sync production-app
```

**Step 3: Inspect the decision with the CLI**
```bash
# Re-login as admin
argocd login <ARGOCD_SERVER> --username admin --password <PASSWORD> --insecure

# Check what permissions dev-user has
argocd account can-i sync applications production/production-app
```

**Expected result:** staging sync is permitted (or returns "app not found"), production sync returns `PermissionDenied`.

---

## Section 7 — Troubleshooting & Best Practices
**Duration: 20 minutes**

### 7.1 Debugging RBAC Denials

**Check ArgoCD server logs**
```bash
kubectl -n argocd logs deployment/argocd-server | grep -i "permission\|rbac\|deny"
```

**Enable debug logging temporarily**
```bash
kubectl -n argocd patch deployment argocd-server \
  --patch '{"spec":{"template":{"spec":{"containers":[{"name":"argocd-server","args":["argocd-server","--loglevel","debug"]}]}}}}'
```

**Test permissions as a specific user (admin only)**
```bash
# Syntax: argocd account can-i <action> <resource> <object>
argocd account can-i sync   applications 'production/my-app'
argocd account can-i delete applications '*//*'
argocd account can-i get    clusters     'https://kubernetes.default.svc'
```

**Inspect the effective policy**
```bash
kubectl -n argocd get configmap argocd-rbac-cm -o yaml
kubectl -n argocd get configmap argocd-cm -o yaml | grep -A5 dex
```

### 7.2 Common Mistakes and Fixes

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| All users get `PermissionDenied` | `policy.default` not set | Set `policy.default: role:readonly` in `argocd-rbac-cm` |
| SSO groups not resolving | Groups claim not in OIDC token | Check IdP group scope; add `groups` scope to connector |
| Wildcard policies not matching | `policy.matchMode` missing | Add `policy.matchMode: glob` to `argocd-rbac-cm` |
| User has group but policy doesn't apply | Wrong group prefix | Check Dex logs for exact claim; match prefix in `g,` rule |
| Admin role not working | Assigned to project role, not global | Global admin = `g, user, role:admin`, not a project role |
| Local user can't log in | Account disabled | Check `accounts.<name>.enabled: "true"` in argocd-cm |
| Deny rule not taking effect | Deny needs explicit `deny` keyword | Use `allow` and `deny` not absence of a rule |
| AppProject blocks sync | Resource kind not whitelisted | Add resource to `namespaceResourceWhitelist` |

### 7.3 Security Best Practices

**Identity & Access**
- Never use `role:admin` for day-to-day operations — create named roles
- Bind roles to SSO groups, not individual users
- Maintain break-glass local admin accounts in a secrets manager
- Rotate API keys on a schedule; audit usage via ArgoCD audit log

**Project Structure**
- Every team gets their own AppProject — never share projects between teams
- Banish the `default` project; all production apps must be in named projects
- Set `sourceRepos` to specific repo patterns, never `*` in production

**Dangerous Actions to Restrict**
- `exec` (pod terminal) — deny for all non-platform roles
- `delete` on `applications` in production — require a separate approval flow
- `override` — limit to senior deployers; it bypasses sync policy

**Sync Windows**
- Enforce change freeze periods via sync windows in AppProjects
- Require `manualSync: false` on production sync windows during freezes

**Audit**
- Enable `audit-log` on the ArgoCD API server
- Ship ArgoCD server logs to your SIEM
- Alert on `role:admin` logins, break-glass account logins, and sync events to production

### 7.4 RBAC Testing Checklist

Before deploying a new RBAC configuration:

- [ ] `policy.default` is set to `role:readonly` (not blank, not admin)
- [ ] All SSO group names match exact claim format from IdP
- [ ] `policy.matchMode: glob` is set if using wildcards
- [ ] Each AppProject has explicit `sourceRepos` and `destinations`
- [ ] `exec` action is explicitly denied for non-platform roles
- [ ] Break-glass account exists and password is stored securely
- [ ] `can-i` tested for each role against its expected permissions
- [ ] Deny rules tested explicitly — absence of allow ≠ deny in all cases
- [ ] Sync windows configured for production projects

---

## Wrap-Up & Q&A
**Duration: 10 minutes**

### Key Takeaways

1. **ArgoCD RBAC is Casbin policy** — `p` for permission, `g` for group assignment, glob matching requires explicit mode
2. **AppProjects are your multi-tenancy boundary** — every team, every environment should have one
3. **SSO groups are the right binding unit** — avoid individual user bindings; they don't scale
4. **Deny explicitly** — especially for `exec`, `delete`, and production sync operations
5. **Two security layers** — ArgoCD RBAC controls ArgoCD actions; Kubernetes RBAC controls what ArgoCD's service account can do; you need both

### What to Do After This Workshop

| Action | Priority | Owner |
|--------|----------|-------|
| Audit current `argocd-rbac-cm` — who has `role:admin`? | High | Platform Team |
| Create AppProjects for each team and migrate apps | High | Platform Team |
| Remove default project usage in production | High | Platform Team |
| Connect SSO and replace local user bindings | Medium | Platform + Security |
| Add explicit deny for `exec` in all non-platform roles | High | Platform Team |
| Enable audit log and ship to SIEM | Medium | Security Team |
| Configure sync windows for production projects | Medium | Platform Team |

---

## Appendices

### Appendix A — Full Policy Reference

```
# Resources
applications       # ArgoCD Applications
clusters           # Registered Kubernetes clusters
repositories       # Git repositories
projects           # AppProjects
accounts           # Local user accounts
certificates       # TLS certificates
gpgkeys            # GPG signing keys
logs               # Pod log access via ArgoCD
exec               # Pod exec/terminal via ArgoCD

# Actions (not all apply to all resources)
get                # Read / view
create             # Create new resource
update             # Modify existing resource
delete             # Remove resource
sync               # Trigger an application sync
override           # Override sync policy settings
action             # Custom resource actions
*                  # All actions (wildcard)
```

### Appendix B — Quick Reference: argocd-rbac-cm Template

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.matchMode: glob
  policy.csv: |
    # Built-in roles are: role:readonly, role:admin

    # --- Custom roles ---
    p, role:viewer,       applications, get,    */*, allow
    p, role:viewer,       projects,     get,    *,   allow

    p, role:deployer,     applications, get,    staging/*, allow
    p, role:deployer,     applications, sync,   staging/*, allow
    p, role:deployer,     applications, override, staging/*, allow
    p, role:deployer,     logs,         get,    staging/*, allow
    p, role:deployer,     exec,         *,      staging/*, deny

    p, role:operator,     applications, get,    */*, allow
    p, role:operator,     applications, sync,   */*, allow
    p, role:operator,     clusters,     get,    *,   allow
    p, role:operator,     repositories, get,    *,   allow
    p, role:operator,     exec,         *,      */*, deny

    # --- Group bindings ---
    g, oidc:readonly-users,     role:viewer
    g, oidc:developers,         role:deployer
    g, oidc:platform-engineers, role:operator
    g, breakglass,              role:admin
```

### Appendix C — Useful ArgoCD CLI Commands

```bash
# Account management
argocd account list
argocd account get --account <name>
argocd account update-password --account <name>
argocd account generate-token --account <name>
argocd account can-i <action> <resource> <object>

# Project management
argocd proj list
argocd proj get <project-name>
argocd proj role list <project-name>
argocd proj role get <project-name> <role-name>
argocd proj role create-token <project-name> <role-name>
argocd proj windows list <project-name>

# App management
argocd app list
argocd app get <app-name>
argocd app sync <app-name>

# Admin
argocd admin settings rbac validate --policy-file policy.csv
argocd admin settings rbac can <role> <action> <resource> <object>
```

### Appendix D — Further Reading

- [ArgoCD RBAC Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/)
- [ArgoCD AppProject Reference](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#projects)
- [Casbin Policy Syntax](https://casbin.org/docs/syntax-for-models)
- [Dex Connector Documentation](https://dexidp.io/docs/connectors/)
- [ArgoCD SSO Configuration](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/)

---

*ArgoCD RBAC Workshop v1.0 | Platform Engineering | For internal use*
