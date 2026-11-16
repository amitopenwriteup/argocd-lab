# Lab: Deploy Kustomize Overlays with the Argo CD UI

*Step-by-step, browser-based lab, about 20 minutes. You deploy the `dev` and `prod` overlays from your GitHub repository using only the Argo CD web interface.*

## Overview

Your repository already contains a Kustomize base and two overlays. In this lab you will create two Argo CD **Applications** from the UI, one per overlay, and see how Argo CD builds, compares and syncs them.

**Repository:** `https://github.com/amitopenwriteup/kustomize-argo.git`

The lab assumes this layout and branch. If yours differs, use your own paths in the forms below.

```
kustomize-argo/
  base/
    kustomization.yaml
    deployment.yaml
    service.yaml
    index.html
  overlays/
    dev/
      kustomization.yaml
      index.html
    prod/
      kustomization.yaml
      resources-patch.yaml
      index.html
```

| Setting | `webapp-dev` | `webapp-prod` |
|---|---|---|
| Overlay path | `overlays/dev` | `overlays/prod` |
| Namespace | `dev` | `prod` |
| Sync policy | Automatic (prune and self-heal) | Manual |
| Replicas | 1 | 3 |
| Image | `nginx:1.25` | `nginx:1.27` |
| Service NodePort | `30081` | `30082` |

**Objectives**

1. Create the dev Application with automatic sync
2. Create the prod Application with manual sync
3. Read the resource tree, diff, events and history
4. Test both applications

## Prerequisites

| Requirement | Details |
|---|---|
| Argo CD | Already installed and running, and you are logged in to the UI as `admin` |
| Kubernetes | Working cluster, `kubectl` available on the control-plane VM |
| Repository | Public, with the layout above pushed to the `main` branch |
| Network | Your browser and `curl` can reach the node IPs on ports `30081` and `30082` |
| Firewall | If `firewalld` is running, NodePorts `30081-30082` are open on the nodes |

---

## Part 1: Create the Dev Application (Automatic Sync)

### Step 1.1 - Start a new app

1. Click **Applications** in the left menu.
2. Click **+ NEW APP**.

### Step 1.2 - General section

| Field | Value |
|---|---|
| Application Name | `webapp-dev` |
| Project Name | `default` |
| Sync Policy | `Automatic` |
| Prune Resources | checked |
| Self Heal | checked |

Under **Sync Options** tick **Auto-Create Namespace**. This makes Argo CD create the `dev` namespace, so you do not need a Namespace manifest in Git.

### Step 1.3 - Source section

| Field | Value |
|---|---|
| Repository URL | `https://github.com/amitopenwriteup/kustomize-argo.git` |
| Revision | `main` |
| Path | `overlays/dev` |

When you enter the path, Argo CD detects the `kustomization.yaml` and shows **Kustomize** settings below the Source section. Leave them unchanged.

### Step 1.4 - Destination section

| Field | Value |
|---|---|
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `dev` |

### Step 1.5 - Create

Click **CREATE** at the top of the form. The `webapp-dev` tile appears on the Applications page.

> Tip: click **EDIT AS YAML** at the top of the form to see the manifest Argo CD generates from your inputs.

### Step 1.6 - Watch the first sync

Because sync is automatic, the tile moves through **Missing**, **Progressing** and finally shows **Healthy** and **Synced** (green). Click the tile to open the application.

---

## Part 2: Create the Prod Application (Manual Sync)

Repeat Part 1 with these values. Click **+ NEW APP** again.

| Field | Value |
|---|---|
| Application Name | `webapp-prod` |
| Project Name | `default` |
| Sync Policy | `Manual` |
| Sync Options | tick **Auto-Create Namespace** |
| Repository URL | `https://github.com/amitopenwriteup/kustomize-argo.git` |
| Revision | `main` |
| Path | `overlays/prod` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `prod` |

Click **CREATE**. The `webapp-prod` tile shows **Missing** and **OutOfSync** (yellow). Nothing is deployed yet because the policy is manual.

### Step 2.1 - Review before syncing

1. Click the `webapp-prod` tile.
2. Click **APP DIFF** in the top toolbar. Every resource is listed as new, because nothing exists in the cluster yet.
3. Click **SYNC**.

### Step 2.2 - Synchronize

In the panel that opens:

1. Leave all resources selected.
2. Leave **Prune** and **Dry Run** unchecked.
3. Click **SYNCHRONIZE**.

Watch the status change to **Syncing**, then **Synced** and **Healthy**.

---

## Part 3: Explore the Application View

Open `webapp-prod` and look at each area.

| Area | What to look for |
|---|---|
| Resource tree | Application, then Service, ConfigMap, Deployment, ReplicaSet and 3 Pods. Names start with `prod-`. |
| Top status bar | Health, Sync status, last sync result and the Git commit that was deployed |
| **DETAILS** | Repository, path, target revision, destination and the **Kustomize** settings (name prefix, namespace, images) |
| **APP DIFF** | Differences between Git and the cluster. Empty when synced. |
| **HISTORY AND ROLLBACK** | Each sync with its commit and time |
| Resource click | Opens a side panel with **SUMMARY**, **EVENTS**, **LOGS** (for Pods) and **LIVE MANIFEST** |
| View buttons | Switch between tree, pods, network and list views |

### Step 3.1 - Check the Kustomize result

1. Click the Deployment `prod-webapp`.
2. Open **LIVE MANIFEST** and confirm `replicas: 3`, `image: nginx:1.27` and the `resources:` block with CPU and memory limits.
3. Click a Pod, open **LOGS** and confirm nginx started.

Repeat for `webapp-dev` and confirm `replicas: 1` and `image: nginx:1.25`.

---

## Part 4: Test the Applications

On the control-plane VM, or from a browser:

```bash
kubectl get all,cm -n dev
kubectl get all,cm -n prod
curl http://<NODE-IP>:30081
curl http://<NODE-IP>:30082
```

Expected output:

```
<h1>Hello from DEV</h1>
<h1>Hello from PROD</h1>
```

Dev has 1 Pod and prod has 3 Pods.

If `curl` cannot connect, check that the NodePorts are open (`sudo firewall-cmd --permanent --add-port=30081-30082/tcp && sudo firewall-cmd --reload`) and that you are using a node's `INTERNAL-IP` from `kubectl get nodes -o wide`.

## Summary

You created two Applications from the Argo CD UI, one per Kustomize overlay. Dev syncs automatically with prune and self-heal, while prod waits for a manual sync after you review the diff. You inspected the resource tree and live manifests, and confirmed with `curl` that each environment serves its own page.
