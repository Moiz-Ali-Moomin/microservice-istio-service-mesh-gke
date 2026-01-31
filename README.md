# Ecommerce Microservice with Istio Ambient Mesh on GKE

This project provides a complete blueprint for deploying a microservice application on Google Kubernetes Engine (GKE) using **Istio Ambient Mesh** (sidecar-less). It features automated infrastructure via Terraform and secure CI/CD using GitHub Actions with OIDC.

---

## 🎯 Why This Project?

This project demonstrates how modern DevOps and Platform Engineering is done in 2026.

It solves common real-world problems:
- Secure CI/CD without storing cloud credentials
- Automated Kubernetes infrastructure on GCP
- Scalable microservice deployment
- Traffic management without sidecar overhead

---

## 🧠 High-Level Architecture

![Architecture Diagram](docs/architecture.png)

1.  **GCP GKE Cluster**: Managed Kubernetes with Workload Identity enabled.
2.  **Istio Ambient Mesh**: High-performance service mesh without sidecar overhead.
3.  **OIDC Authentication**: Secure, keyless authentication between GitHub and GCP.
4.  **GitHub Actions**: Fully automated deployment on every push to `main`.

---



## 🧭 How to Use This Project

### Beginner Path
If you are new to DevOps or Kubernetes:
1. Read the Architecture section
2. Skim Terraform code (don’t try to understand everything)
3. Focus on how CI/CD deploys to GKE
4. Learn what Istio Ambient Mesh does conceptually

### Advanced Path
If you are experienced:
1. Review OIDC Workload Identity Federation
2. Inspect IAM least-privilege policies
3. Study Istio Ambient Mesh (ztunnel + waypoint)
4. Extend with canary or traffic shifting

## 📚 What You Will Learn From This Project

- How GitHub Actions authenticates to GCP using OIDC
- How to provision GKE securely using Terraform
- How Istio Ambient Mesh works without sidecars
- How CI/CD deploys microservices into Kubernetes
- How modern platform teams reduce operational overhead

---

## 📋 Prerequisites
- A Google Cloud Platform account and a project.
- A GitHub repository.
- Local tools: `gcloud`, `terraform`, `kubectl`.

---

## 🛠️ Step 1: Configure GCP Project
Before running Terraform, ensure your GCP project is ready:

1.  **Enable Required APIs**:
    ```bash
    gcloud services enable \
      compute.googleapis.com \
      container.googleapis.com \
      iam.googleapis.com \
      iamcredentials.googleapis.com \
      sts.googleapis.com
    ```
2.  **Set your Project ID**:
    ```bash
    gcloud config set project YOUR_PROJECT_ID
    ```

---

## 🏗️ Step 2: Infrastructure with Terraform
This step creates the VPC, GKE cluster, and OIDC Workload Identity Federation.

1.  **Navigate to Terraform Directory**:
    ```bash
    cd terraform
    ```
2.  **Initialize and Apply**:
    ```bash
    terraform init
    terraform apply \
      -var="project_id=YOUR_PROJECT_ID" \
      -var="github_repo=YOUR_GITHUB_REPO" # e.g., "myuser/my-repo"
    ```
3.  **Save the Outputs**:
    Terraform will output `WIF_PROVIDER` and `WIF_SERVICE_ACCOUNT`. **Copy these.**

---

## 🔐 Step 3: Setup GitHub Actions & OIDC
We use OIDC (Workload Identity Federation) to avoid using static long-lived JSON keys.

1.  **Go to your GitHub Repository** -> `Settings` -> `Secrets and variables` -> `Actions`.
2.  **Add the following Repository Secrets**:
    - `GCP_PROJECT_ID`: Your GCP project ID.
    - `WIF_PROVIDER`: The output from Terraform (e.g., `projects/12345/locations/global/workloadIdentityPools/github-pool/providers/github-provider`).
    - `WIF_SERVICE_ACCOUNT`: The service account email from Terraform (e.g., `github-actions-sa@yourproject.iam.gserviceaccount.com`).

---

## 🛳️ Step 4: The Deployment Process
Once you push code to the `main` branch, the GitHub Action (`.github/workflows/ci.yml`) triggers:

### Flow:
1.  **GCP Auth**: Uses the `google-github-actions/auth` action to exchange a GitHub token for a short-lived GCP access token using OIDC.
2.  **Connect to GKE**: Pulls the cluster credentials.
3.  **Istio Installation**:
    - Downloads Istio CLI.
    - Installs Istio using the **Ambient profile**:
      ```bash
      istioctl install --set profile=ambient -y
      ```
4.  **Namespace Configuration**:
    - Creates the `ecommerce` namespace.
    - Labels it for Ambient mode: `istio.io/dataplane-mode=ambient`.
5.  **Service Mesh Deployment**:
    - Deploys the **Waypoint Proxy** (the L7 engine for our sidecar-less mesh).
    - Deploys the **Gateway API** resources for ingress traffic.
6.  **Microservices**: Deploys all ecommerce apps found in `microservices/` folder.

---

## 🔍 Step 5: Verify Your Mesh
After the pipeline finishes, run these commands to see your Sidecar-less mesh in action:

1.  **Check Pods (No Sidecars)**:
    ```bash
    kubectl get pods -n ecommerce
    # Note: READY should show "1/1", showing NO sidecar proxy container!
    ```
2.  **Check L4 Connectivity**:
    ```bash
    istioctl proxy-status
    # Shows the node-shared ztunnel handles the security.
    ```
3.  **Check Waypoint (L7 Proxy)**:
    ```bash
    kubectl get gtw -n ecommerce
    # Verify the ecommerce-waypoint is programmed.
    ```

---



## 🧠 Key Design Decisions

- Used OIDC instead of service account keys to improve security
- Chose Istio Ambient Mesh to reduce sidecar overhead
- Terraform manages both infrastructure and IAM for consistency
- GitHub Actions used instead of Jenkins for cloud-native CI/CD

## 🛠️ Common Issues & Fixes

### GitHub Actions authentication fails
- Ensure WIF_PROVIDER and WIF_SERVICE_ACCOUNT secrets are correct
- Verify GitHub repo name matches Terraform configuration

### Pods not joining the mesh
- Check namespace label: istio.io/dataplane-mode=ambient
- Ensure ztunnel is running in istio-system

### No ingress traffic
- Verify Gateway and HTTPRoute resources
- Check waypoint proxy status

## 💼 How This Project Maps to Real DevOps Roles

- DevOps Engineer → CI/CD, Kubernetes, Terraform
- Cloud Engineer → GCP, IAM, Workload Identity
- Platform Engineer → Service Mesh, secure automation

## 🚧 Future Improvements

- Canary deployments using Istio traffic splitting
- GitOps with ArgoCD
- HPA based on custom metrics
- Distributed tracing with OpenTelemetry

---

## 🧹 Cleanup
To avoid GCP costs, destroy everything when done:
```bash
cd terraform
terraform destroy -var="project_id=YOUR_PROJECT_ID" -var="github_repo=YOUR_GITHUB_REPO"
```
