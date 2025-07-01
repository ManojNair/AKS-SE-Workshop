# Exercise 1: AKS Cluster Fundamentals

## Overview

In this exercise, you will create your first Azure Kubernetes Service (AKS) cluster and learn the fundamentals of AKS architecture and management.

## Learning Objectives

By the end of this exercise, you will be able to:
- Create an AKS cluster using Azure CLI
- Understand AKS architecture components
- Perform basic cluster management operations
- Connect to your cluster using kubectl
- Verify cluster health and status

## Prerequisites

- Completed workshop prerequisites installation
- Authenticated with Azure CLI
- Active Azure subscription

## Exercise Steps

### Step 1: Authentication and Setup

First, ensure you're authenticated and set up your environment:

#### Windows (PowerShell)
```powershell
# Login to Azure
az login

# Set your subscription (if you have multiple)
az account set --subscription "Your-Subscription-Name-or-ID"

# Verify current subscription
az account show
```

#### macOS/Linux
```bash
# Login to Azure
az login

# Set your subscription (if you have multiple)
az account set --subscription "Your-Subscription-Name-or-ID"

# Verify current subscription
az account show
```

### Step 2: Create Resource Group

Create a resource group to organize your AKS resources:

#### Windows (PowerShell)
```powershell
# Create resource group
az group create `
  --name "rg-aks-workshop-01" `
  --location "Australia East" `
  --tags Environment=Workshop Purpose=Learning Exercise=01
```

#### macOS/Linux
```bash
# Create resource group
az group create \
  --name "rg-aks-workshop-01" \
  --location "Australia East" \
  --tags Environment=Workshop Purpose=Learning Exercise=01
```

### Step 3: Create Virtual Network (Optional but Recommended)

For better network control and isolation:

#### Windows (PowerShell)
```powershell
# Create virtual network
az network vnet create `
  --resource-group "rg-aks-workshop-01" `
  --name "vnet-aks-workshop-01" `
  --address-prefix "10.0.0.0/16" `
  --subnet-name "aks-subnet" `
  --subnet-prefix "10.0.1.0/24"

# Get subnet ID for AKS
$SUBNET_ID = az network vnet subnet show `
  --resource-group "rg-aks-workshop-01" `
  --vnet-name "vnet-aks-workshop-01" `
  --name "aks-subnet" `
  --query id -o tsv
```

#### macOS/Linux
```bash
# Create virtual network
az network vnet create \
  --resource-group "rg-aks-workshop-01" \
  --name "vnet-aks-workshop-01" \
  --address-prefix "10.0.0.0/16" \
  --subnet-name "aks-subnet" \
  --subnet-prefix "10.0.1.0/24"

# Get subnet ID for AKS
SUBNET_ID=$(az network vnet subnet show \
  --resource-group "rg-aks-workshop-01" \
  --vnet-name "vnet-aks-workshop-01" \
  --name "aks-subnet" \
  --query id -o tsv)
```

### Step 4: Create AKS Cluster

Create your first AKS cluster with development-optimized settings:

#### Windows (PowerShell)
```powershell
# Create AKS cluster with custom networking
az aks create `
  --resource-group "rg-aks-workshop-01" `
  --name "aks-workshop-01-cluster" `
  --node-count 2 `
  --node-vm-size "Standard_B2s" `
  --enable-managed-identity `
  --network-plugin azure `
  --vnet-subnet-id $SUBNET_ID `
  --service-cidr "10.1.0.0/16" `
  --dns-service-ip "10.1.0.10" `
  --docker-bridge-address "172.17.0.1/16" `
  --generate-ssh-keys `
  --tags Environment=Workshop Purpose=Learning Exercise=01 `
  --enable-addons monitoring `
  --workspace-resource-id "/subscriptions/YOUR-SUBSCRIPTION-ID/resourceGroups/rg-aks-workshop-01/providers/Microsoft.OperationalInsights/workspaces/loganalytics-workspace"
```

#### macOS/Linux
```bash
# Create AKS cluster with custom networking
az aks create \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --node-count 2 \
  --node-vm-size "Standard_B2s" \
  --enable-managed-identity \
  --network-plugin azure \
  --vnet-subnet-id "$SUBNET_ID" \
  --service-cidr "10.1.0.0/16" \
  --dns-service-ip "10.1.0.10" \
  --docker-bridge-address "172.17.0.1/16" \
  --generate-ssh-keys \
  --tags Environment=Workshop Purpose=Learning Exercise=01 \
  --enable-addons monitoring \
  --workspace-resource-id "/subscriptions/YOUR-SUBSCRIPTION-ID/resourceGroups/rg-aks-workshop-01/providers/Microsoft.OperationalInsights/workspaces/loganalytics-workspace"
```

**Note**: Replace `YOUR-SUBSCRIPTION-ID` with your actual subscription ID. You can get this by running `az account show --query id -o tsv`.

### Step 5: Get Cluster Credentials

Download the cluster credentials to connect with kubectl:

#### Windows (PowerShell)
```powershell
# Get credentials for kubectl
az aks get-credentials `
  --resource-group "rg-aks-workshop-01" `
  --name "aks-workshop-01-cluster" `
  --overwrite-existing
```

#### macOS/Linux
```bash
# Get credentials for kubectl
az aks get-credentials \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --overwrite-existing
```

### Step 6: Verify Cluster Setup

Verify that your cluster is running correctly:

```bash
# Check cluster status
az aks show \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --query provisioningState

# Verify kubectl connection
kubectl cluster-info

# Check nodes
kubectl get nodes

# Check system pods
kubectl get pods --all-namespaces

# Check cluster version
kubectl version --short
```

### Step 7: Explore AKS Architecture

Understand the components of your AKS cluster:

```bash
# View cluster details
az aks show \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --output table

# Check node pool information
az aks nodepool list \
  --resource-group "rg-aks-workshop-01" \
  --cluster-name "aks-workshop-01-cluster"

# View network profile
az aks show \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --query networkProfile
```

### Step 8: Basic Cluster Management

Perform basic management operations:

```bash
# List all contexts
kubectl config get-contexts

# Set the AKS context as default
kubectl config use-context aks-workshop-01-cluster

# Verify current context
kubectl config current-context

# Check cluster capacity
kubectl top nodes

# Check pod resource usage
kubectl top pods --all-namespaces
```

## Understanding AKS Architecture

### Key Components

1. **Control Plane (Managed by Azure)**
   - API Server
   - etcd database
   - Scheduler
   - Controller Manager

2. **Node Pools**
   - Worker nodes running your applications
   - Each node is a VM in Azure
   - Can have multiple node pools with different configurations

3. **Networking**
   - Azure CNI for pod networking
   - Load balancers for service exposure
   - Network security groups for traffic control

4. **Storage**
   - Azure Disks for persistent storage
   - Azure Files for shared storage
   - Azure Blob Storage for object storage

### Exercise Questions

Answer these questions to reinforce your learning:

1. **What is the difference between the control plane and node pools in AKS?**
   - Control plane is managed by Azure and handles cluster orchestration
   - Node pools contain worker nodes where your applications run

2. **Why do we use Standard_B2s VM size for this workshop?**
   - Cost-effective for learning and development
   - Sufficient resources for basic workloads
   - Good balance of CPU, memory, and cost

3. **What is the purpose of the virtual network in AKS?**
   - Provides network isolation
   - Enables custom routing and security
   - Allows integration with other Azure services

## Hands-on Challenges

### Challenge 1: Scale Your Cluster
Scale your cluster to 3 nodes and then back to 2:

```bash
# Scale up to 3 nodes
az aks scale \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --node-count 3

# Verify scaling
kubectl get nodes

# Scale back to 2 nodes
az aks scale \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --node-count 2
```

### Challenge 2: Explore Cluster Resources
Create a namespace and deploy a simple application:

```bash
# Create a namespace
kubectl create namespace workshop-demo

# Deploy a simple nginx pod
kubectl run nginx-demo \
  --image=nginx:latest \
  --port=80 \
  --namespace=workshop-demo

# Check the pod status
kubectl get pods --namespace=workshop-demo

# Describe the pod to see details
kubectl describe pod nginx-demo --namespace=workshop-demo
```

### Challenge 3: Monitor Cluster Health
Use Azure Monitor to view cluster metrics:

```bash
# Open Azure Monitor for containers
az aks browse \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster"
```

## Troubleshooting

### Common Issues

1. **Authentication Issues**
   ```bash
   # Re-authenticate with Azure
   az login
   az account set --subscription "Your-Subscription-Name-or-ID"
   ```

2. **kubectl Connection Issues**
   ```bash
   # Refresh cluster credentials
   az aks get-credentials \
     --resource-group "rg-aks-workshop-01" \
     --name "aks-workshop-01-cluster" \
     --overwrite-existing
   ```

3. **Resource Group Not Found**
   ```bash
   # List resource groups
   az group list --output table
   
   # Create resource group if missing
   az group create --name "rg-aks-workshop-01" --location "Australia East"
   ```

## Cleanup

When you're done with this exercise, clean up resources to avoid unnecessary costs:

```bash
# Delete the AKS cluster
az aks delete \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --yes

# Delete the resource group (this will delete all resources)
az group delete \
  --name "rg-aks-workshop-01" \
  --yes
```

## Summary

In this exercise, you have:
- ✅ Created your first AKS cluster
- ✅ Understood AKS architecture components
- ✅ Connected to your cluster using kubectl
- ✅ Performed basic cluster management operations
- ✅ Explored cluster resources and monitoring

## Next Steps

You're now ready to move to **Exercise 2: Application Deployment and Management** where you'll learn how to deploy and manage applications in your AKS cluster.

## Additional Resources

- [AKS Architecture](https://docs.microsoft.com/en-us/azure/aks/concepts-clusters-workloads)
- [AKS Networking](https://docs.microsoft.com/en-us/azure/aks/concepts-network)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [AKS Best Practices](https://docs.microsoft.com/en-us/azure/aks/best-practices) 