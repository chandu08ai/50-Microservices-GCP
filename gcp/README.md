# RailShack · Deploying 50 Microservices on Google Cloud Platform (GCP)

This guide provides end-to-end instructions for deploying the **RailShack 50-Microservices Project** on **Google Cloud Platform (GCP)** using **Google Kubernetes Engine (GKE)** and **Helm**.

---

## 🏗 Architecture & Cloud Mapping (AWS vs. GCP)

The original project was authored for AWS EKS. Here is how every cloud component maps to Google Cloud Platform:

| Component | AWS Implementation | GCP Equivalent | Details |
|---|---|---|---|
| **Kubernetes Engine** | AWS EKS (Auto Mode) | **GCP GKE** (Google Kubernetes Engine) | Managed Kubernetes cluster running all 50 microservices. |
| **External Traffic** | AWS ALB (Application Load Balancer) | **GCP Network Load Balancer** (`Service type: LoadBalancer`) | Provides a public IPv4 address to reach the frontend UI. |
| **Database Storage** | AWS EBS (`ebs.csi.eks.amazonaws.com`, gp3) | **GCP Persistent Disk** (`pd.csi.storage.gke.io`, `standard-rwo`) | Keeps PostgreSQL state persistent across restarts. |
| **CLI Tool** | `aws` / `eksctl` | `gcloud` | Command-line management tool for GCP resources. |
| **Deployment** | Helm 3 | **Helm 3** | Cloud-agnostic package manager. |

---

## 📋 Prerequisites

Make sure the following command-line tools are installed on your workstation:

1. **Google Cloud SDK (`gcloud` CLI):**
   ```powershell
   winget install Google.CloudSDK
   ```
2. **Kubernetes CLI (`kubectl`):**
   ```powershell
   winget install Kubernetes.kubectl
   ```
3. **Helm 3:**
   ```powershell
   winget install Helm.Helm
   ```
4. **GKE Authentication Plugin (Mandatory for GKE):**
   ```powershell
   gcloud components install gke-gcloud-auth-plugin
   ```

Verify your local environment:
```powershell
gcloud --version
kubectl version --client
helm version
```

---

## 🚀 Step-by-Step Deployment Guide

### Step 1: Authenticate with Google Cloud

1. Log in to your Google Cloud account:
   ```powershell
   gcloud auth login
   ```
2. Set your active Google Cloud project:
   ```powershell
   # List your projects
   gcloud projects list

   # Set project
   gcloud config set project <YOUR_PROJECT_ID>
   ```
3. Enable the required APIs (Kubernetes Engine & Compute Engine):
   ```powershell
   gcloud services enable container.googleapis.com compute.googleapis.com
   ```

---

### Step 2: Review GCP Values File (`gcp/values-gcp.yaml`)

This repository contains [`gcp/values-gcp.yaml`](values-gcp.yaml), which customizes the Helm chart specifically for GCP:

```yaml
app:
  environment: production
  clusterName: gcp-gke

defaults:
  replicas: 1              # 1 pod per service fits easily in a standard budget/trial
  pdb:
    enabled: false         # Disabling PDB avoids node draining blocks during testing

# Expose frontend via standard GCP external LoadBalancer
access:
  mode: loadbalancer       # In GKE, this automatically provisions a Google Cloud External IP!

postgres:
  storage:
    className: standard-rwo  # GKE's default Persistent Disk CSI driver
    createEBSClass: false    # Turn OFF AWS EBS StorageClass creation
  metrics:
    enabled: false

redis:
  metrics:
    enabled: false

serviceMonitor:
  enabled: false
grafanaDashboard:
  enabled: false
prometheusRules:
  enabled: false
grafanaIngress:
  enabled: false
```

---

### Step 3: Create the GKE Cluster

Run the following command to provision a single-zone, cost-effective GKE cluster:

```powershell
gcloud container clusters create railshack-cluster `
  --zone us-central1-a `
  --num-nodes 2 `
  --machine-type e2-standard-4 `
  --disk-size 50 `
  --disk-type pd-standard
```

- **Zone:** `us-central1-a` keeps all nodes in one zone, eliminating inter-zone networking fees.
- **Node Size:** 2 nodes of `e2-standard-4` provide 8 vCPUs and 32 GB RAM total, easily hosting all 50 microservices + Postgres + Redis.

---

### Step 4: Connect `kubectl` to the Cluster

Fetch cluster credentials into your local kubeconfig:

```powershell
gcloud container clusters get-credentials railshack-cluster --zone us-central1-a
```

Verify your cluster connectivity:
```powershell
kubectl get nodes
```
*(Both nodes should report `Ready` status).*

---

### Step 5: Deploy RailShack with Helm

Run Helm from the root of the repository:

```powershell
helm upgrade --install railshack helm/railshack `
  -n railshack `
  --create-namespace `
  -f gcp/values-gcp.yaml `
  --set image.repository=devopsshack `
  --set image.tag=latest `
  --wait --timeout 15m
```

> **Note:** If you build your own images, replace `--set image.repository=devopsshack` with your GitHub username or Google Artifact Registry path.

---

### Step 6: Verify Deployment and Retrieve External IP

1. **Check Pod status:**
   ```powershell
   kubectl get pods -n railshack
   ```
   *Within 2 to 3 minutes, all 52 pods should show `Running`.*

2. **Retrieve the Public IP address:**
   ```powershell
   kubectl get svc frontend -n railshack
   ```
   Look for the `EXTERNAL-IP` value.

3. **Open the web application:**
   Navigate in your browser to:
   ```
   http://<EXTERNAL_IP>
   ```
   - **Demo Login:** Username: `devopsshack` | Password: `devopsshack`

4. **Run the Automated Smoke Test:**
   ```powershell
   .\scripts\smoke-test.ps1 "http://<EXTERNAL_IP>"
   ```

---

## 🧹 Cost Cleanup (Deleting Resources)

When you are done testing, destroy the resources so you do not incur ongoing cloud costs:

```powershell
# 1. Uninstall the Helm release
helm uninstall railshack -n railshack

# 2. Delete the PostgreSQL persistent volume claim
kubectl delete pvc data-postgres-0 -n railshack

# 3. Delete the GKE cluster and associated VMs / load balancers
gcloud container clusters delete railshack-cluster --zone us-central1-a --quiet
```

---

## 🔧 Troubleshooting

- **`gke-gcloud-auth-plugin` missing:**
  Run `gcloud components install gke-gcloud-auth-plugin` and restart your terminal.
- **Pods stuck in `Pending`:**
  Check cluster capacity with `kubectl describe pod <pod-name> -n railshack`. Ensure your machine type has at least 8 vCPUs across the cluster.
- **External IP shows `<pending>`:**
  GCP Load Balancers take about 1–2 minutes to provision. Wait a short moment and re-run `kubectl get svc frontend -n railshack`.
