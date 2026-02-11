# Ecommerce Microservice with Istio Service Mesh on GKE

This project provides a complete blueprint for deploying a microservice application on Google Kubernetes Engine (GKE) or any Kubernetes cluster using **Istio Service Mesh**. It features automated infrastructure via Terraform and secure CI/CD using GitHub Actions.

---

## 🎯 Architecture

![Architecture Diagram](docs/architecture.png)

1.  **Kubernetes Cluster**: Managed GKE or standard Kubernetes.
2.  **Istio Service Mesh**: Sidecar-based mesh for traffic management, security, and observability.
3.  **Microservices**: E-commerce microservices (UI, Order, Product, etc.).
4.  **Ingress Gateway**: Istio Gateway managing entry traffic.

---

## 🛠️ Prerequisites

- **Kubernetes Cluster** (GKE, Minikube, Docker Desktop, etc.)
- **Tools**:
    - `kubectl`
    - `helm`
    - `curl`

---

## 🚀 Installation Guide

This guide follows the standard Helm-based installation for Istio.

### 1. Install Istio (Base & Istiod)

Add the Istio Helm repository:

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

Create the `istio-system` namespace and install the core components:

```bash
kubectl create namespace istio-system

# Install Istio Base
helm install istio-base istio/base -n istio-system --version 1.25.2

# Install Istiod (Control Plane)
helm install istiod istio/istiod -n istio-system --version 1.25.2 --wait
```

### 2. Install Istio Ingress Gateway

Install the Gateway to manage incoming traffic:

```bash
helm install istio-ingress istio/gateway -n istio-system --version 1.24.0
```

### 3. Install Add-ons (Observability)

Install Prometheus, Grafana, Kiali, and Jaeger for monitoring and tracing:

```bash
mkdir -p istio-addons

# Download manifests
curl -L https://raw.githubusercontent.com/istio/istio/release-1.25/samples/addons/prometheus.yaml -o istio-addons/prometheus.yaml
curl -L https://raw.githubusercontent.com/istio/istio/release-1.25/samples/addons/grafana.yaml -o istio-addons/grafana.yaml
curl -L https://raw.githubusercontent.com/istio/istio/release-1.25/samples/addons/kiali.yaml -o istio-addons/kiali.yaml
curl -L https://raw.githubusercontent.com/istio/istio/release-1.25/samples/addons/jaeger.yaml -o istio-addons/jaeger.yaml

# Apply manifests
kubectl apply -f istio-addons/prometheus.yaml
kubectl apply -f istio-addons/grafana.yaml
kubectl apply -f istio-addons/kiali.yaml
kubectl apply -f istio-addons/jaeger.yaml
```

Verifying the installation:
```bash
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```

### 4. Deploy Application Namespace

Create the `ecommerce` namespace and enable **Istio Sidecar Injection**:

```bash
kubectl create namespace ecommerce
kubectl label namespace ecommerce istio-injection=enabled
```

### 5. Deploy Microservices

Deploy the configuration maps and microservices:

```bash
# Apply ConfigMap
kubectl apply -f microservices/configs.yaml

# Apply all microservices
kubectl apply -f microservices/
```

### 6. Apply Istio Traffic Rules

Apply the Gateway and VirtualService configurations:

```bash
kubectl apply -f istio/
```

---

## 🔍 Verification

### Check Pods
Ensure all pods are running and have **2/2** containers (application + istio-proxy sidecar):

```bash
kubectl get pods -n ecommerce
# Example output:
# NAME                               READY   STATUS    RESTARTS   AGE
# ecommerce-ui-xxx                   2/2     Running   0          1m
# product-catalog-xxx                2/2     Running   0          1m
```

### Access Application
Get the External IP of the Ingress Gateway:

```bash
kubectl get svc -n istio-system istio-ingress
```
Access the application at `http://<EXTERNAL-IP>/`.

### Observability Dashboard (Kiali)
To visualize the mesh:

```bash
istioctl dashboard kiali
# OR manually:
kubectl port-forward svc/kiali -n istio-system 20001:20001
# Open http://localhost:20001
```

---

## 🧪 Advanced Features

### Canary Deployment
Deploy a canary version of `product-catalog`:

```bash
kubectl apply -f canary-deployment/product-catalog-v2.yaml
```

### Circuit Breaker
Deploy a circuit breaker example:

```bash
kubectl apply -f circuit-breaker-func/order-management-v2.yaml
```

---

## 🧹 Cleanup

To remove the deployment:

```bash
kubectl delete -f microservices/
kubectl delete namespace ecommerce
helm uninstall istio-ingress -n istio-system
helm uninstall istiod -n istio-system
helm uninstall istio-base -n istio-system
kubectl delete namespace istio-system
```
