# AKS CI/CD + Managed Prometheus/Grafana (Dev)

This guide sets up Azure DevOps CI/CD, Azure Container Registry (ACR), Azure Kubernetes Service (AKS), and Azure Managed Prometheus/Grafana for a dev environment sized for ~10 users.

## Architecture

- Azure DevOps Pipeline builds and pushes the Docker image to ACR.
- AKS pulls the image and runs the deployment in the `dev` namespace.
- Azure Managed Prometheus/Grafana monitors AKS and the workload.

## Prerequisites

- Azure subscription with **Owner** or **Contributor** access.
- Azure DevOps project with permissions to create service connections.
- Azure CLI, kubectl, and Helm installed locally (for provisioning).

## Suggested Sizing (Dev / ~10 Users)

- AKS node pool: **2x Standard_B2s** (2 vCPU, 4 GB RAM each)
- Kubernetes deployment replicas: **2** (defined in `k8s/deployment.yaml`)

## Provisioning (Example)

> Replace names and regions as needed.

```bash
# Resource group
az group create --name rg-angular-nodejs-dev --location eastus

# Container registry
az acr create --resource-group rg-angular-nodejs-dev --name myacr --sku Basic

# AKS cluster
az aks create \
  --resource-group rg-angular-nodejs-dev \
  --name aks-angular-nodejs-dev \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --enable-managed-identity \
  --attach-acr myacr

# Fetch kubeconfig
az aks get-credentials --resource-group rg-angular-nodejs-dev --name aks-angular-nodejs-dev
```

## Enable Azure Managed Prometheus/Grafana

```bash
# Enable Azure Monitor for containers (includes managed Prometheus)
az aks enable-addons \
  --resource-group rg-angular-nodejs-dev \
  --name aks-angular-nodejs-dev \
  --addons monitoring

# Create a Managed Grafana instance
az grafana create \
  --name grafana-angular-nodejs-dev \
  --resource-group rg-angular-nodejs-dev
```

> In Grafana, connect to the managed Prometheus workspace created by the AKS monitoring addon.

## Azure DevOps Service Connections

Create the following service connections:

1. **ACR Service Connection**
   - Type: Docker Registry
   - Registry: `myacr.azurecr.io`
   - Name used in pipeline: `ACR-SERVICE-CONNECTION`

2. **AKS Service Connection**
   - Type: Kubernetes
   - Cluster: `aks-angular-nodejs-dev`
   - Namespace: `dev`
   - Name used in pipeline: `AKS-SERVICE-CONNECTION`

Update the values in `azure-pipelines.yml` if you choose different names.

## Pipeline Overview

- **Build stage**: Docker build and push to ACR.
- **Deploy stage**: Apply namespace, deployment, service, ingress, and set image tag.

## Kubernetes Manifests

- `k8s/namespace.yaml`
- `k8s/deployment.yaml`
- `k8s/service.yaml`
- `k8s/ingress.yaml`

### Update the ingress host

Set your DNS host in `k8s/ingress.yaml`:

```yaml
host: angular-nodejs-example.dev.example.com
```

If you are using a public domain, create a DNS A record pointing to the AKS ingress public IP.

## Optional Enhancements

- Add an HPA for CPU-based autoscaling.
- Add TLS with cert-manager for HTTPS.
- Add Azure Monitor alerts for latency and error rate.
