# Exercise 9: Node Auto Provisioner (NAP) with Karpenter

## Overview

This exercise covers the implementation of Node Auto Provisioner (NAP) using Karpenter in Azure Kubernetes Service (AKS). You'll learn how to set up dynamic node provisioning, automatic scaling, and cost optimization for your AKS clusters.

## Learning Objectives

- Understand Node Auto Provisioner concepts and benefits
- Install and configure Karpenter in AKS
- Create NodePools and Provisioners for different workload types
- Implement cost optimization strategies
- Monitor and troubleshoot dynamic node provisioning
- Set up multi-zone and spot instance configurations

## Prerequisites

- Completed Exercises 1-8
- Azure subscription with appropriate permissions
- Azure CLI installed and configured
- kubectl installed
- Helm installed
- Understanding of Kubernetes node pools and scaling

## Duration

- **Estimated Time**: 2-3 hours
- **Difficulty**: Advanced

---

## Part 1: Understanding Node Auto Provisioner (NAP)

### 1.1 What is Node Auto Provisioner?

Node Auto Provisioner (NAP) is a Kubernetes-native solution that automatically provisions nodes based on pod requirements. It eliminates the need for manual node pool management and provides:

**Key Benefits:**
- **Just-in-time Provisioning**: Nodes are created only when needed
- **Cost Optimization**: Automatic scaling down when resources are not needed
- **Workload Optimization**: Right-sizing nodes for specific workloads
- **Multi-zone Support**: Automatic distribution across availability zones
- **Spot Instance Integration**: Cost savings with spot/preemptible instances

### 1.2 Karpenter vs Cluster Autoscaler

| Feature | Karpenter | Cluster Autoscaler |
|---------|-----------|-------------------|
| **Node Provisioning** | Dynamic, just-in-time | Pre-existing node pools |
| **Node Types** | Any instance type | Limited to node pool types |
| **Scaling Speed** | Seconds to minutes | Minutes to tens of minutes |
| **Cost Optimization** | Advanced bin-packing | Basic scaling |
| **Multi-cloud** | Cloud-agnostic | Cloud-specific |

### 1.3 AKS NAP Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Pod Request   │───▶│   Karpenter     │───▶│   Azure VMSS    │
│   (CPU/Memory)  │    │   Controller    │    │   Provisioning  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                              │
                              ▼
                       ┌─────────────────┐
                       │   Node Ready    │
                       │   Pod Scheduled │
                       └─────────────────┘
```

---

## Part 2: Prerequisites and Environment Setup

### 2.1 Enable Node Auto Provisioner

```bash
# Check if NAP is enabled on your AKS cluster
az aks show \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --query "autoScalerProfile.enableNodeAutoprovisioning"

# Enable NAP if not already enabled
az aks update \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --enable-node-autoprovisioning \
  --node-autoprovisioning-config "min-count=1,max-count=10,scale-down-delay-after-add=10m,scale-down-delay-after-delete=10s,scale-down-delay-after-failure=3m,scan-interval=10s,scale-down-unneeded=10m"

# Verify NAP is enabled
az aks show \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --query "autoScalerProfile"
```

**Windows PowerShell:**
```powershell
# Check if NAP is enabled on your AKS cluster
az aks show `
  --resource-group "rg-aks-workshop-01" `
  --name "aks-workshop-01-cluster" `
  --query "autoScalerProfile.enableNodeAutoprovisioning"

# Enable NAP if not already enabled
az aks update `
  --resource-group "rg-aks-workshop-01" `
  --name "aks-workshop-01-cluster" `
  --enable-node-autoprovisioning `
  --node-autoprovisioning-config "min-count=1,max-count=10,scale-down-delay-after-add=10m,scale-down-delay-after-delete=10s,scale-down-delay-after-failure=3m,scan-interval=10s,scale-down-unneeded=10m"

# Verify NAP is enabled
az aks show `
  --resource-group "rg-aks-workshop-01" `
  --name "aks-workshop-01-cluster" `
  --query "autoScalerProfile"
```

### 2.2 Install Karpenter

```bash
# Add Karpenter Helm repository
helm repo add karpenter https://charts.karpenter.sh
helm repo update

# Create namespace for Karpenter
kubectl create namespace karpenter

# Install Karpenter
helm install karpenter karpenter/karpenter \
  --namespace karpenter \
  --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="" \
  --set aws.enabled=false \
  --set azure.enabled=true

# Verify Karpenter installation
kubectl get pods -n karpenter
kubectl get crd | grep karpenter
```

**Windows PowerShell:**
```powershell
# Add Karpenter Helm repository
helm repo add karpenter https://charts.karpenter.sh
helm repo update

# Create namespace for Karpenter
kubectl create namespace karpenter

# Install Karpenter
helm install karpenter karpenter/karpenter `
  --namespace karpenter `
  --create-namespace `
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="" `
  --set aws.enabled=false `
  --set azure.enabled=true

# Verify Karpenter installation
kubectl get pods -n karpenter
kubectl get crd | grep karpenter
```

### 2.3 Configure Azure Identity for Karpenter

```bash
# Create managed identity for Karpenter
az identity create \
  --resource-group "rg-aks-workshop-01" \
  --name "karpenter-identity"

# Get identity details
IDENTITY_ID=$(az identity show \
  --resource-group "rg-aks-workshop-01" \
  --name "karpenter-identity" \
  --query id -o tsv)

IDENTITY_CLIENT_ID=$(az identity show \
  --resource-group "rg-aks-workshop-01" \
  --name "karpenter-identity" \
  --query clientId -o tsv)

echo "Identity ID: $IDENTITY_ID"
echo "Client ID: $IDENTITY_CLIENT_ID"

# Assign permissions to the managed identity
az role assignment create \
  --assignee $IDENTITY_CLIENT_ID \
  --role "Virtual Machine Contributor" \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-aks-workshop-01"

az role assignment create \
  --assignee $IDENTITY_CLIENT_ID \
  --role "Network Contributor" \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-aks-workshop-01"
```

**Windows PowerShell:**
```powershell
# Create managed identity for Karpenter
az identity create `
  --resource-group "rg-aks-workshop-01" `
  --name "karpenter-identity"

# Get identity details
$IDENTITY_ID = az identity show `
  --resource-group "rg-aks-workshop-01" `
  --name "karpenter-identity" `
  --query id -o tsv

$IDENTITY_CLIENT_ID = az identity show `
  --resource-group "rg-aks-workshop-01" `
  --name "karpenter-identity" `
  --query clientId -o tsv

Write-Host "Identity ID: $IDENTITY_ID"
Write-Host "Client ID: $IDENTITY_CLIENT_ID"

# Assign permissions to the managed identity
az role assignment create `
  --assignee $IDENTITY_CLIENT_ID `
  --role "Virtual Machine Contributor" `
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-aks-workshop-01"

az role assignment create `
  --assignee $IDENTITY_CLIENT_ID `
  --role "Network Contributor" `
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-aks-workshop-01"
```

---

## Part 3: Creating NodePools and Provisioners

### 3.1 Create NodePool for General Workloads

```bash
# Create NodePool for general workloads
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: NodePool
metadata:
  name: general
spec:
  disruption:
    consolidationPolicy: WhenEmpty
    consolidationTTL: 30s
  nodeClassRef:
    apiVersion: karpenter.k8s.aws/v1beta1
    kind: EC2NodeClass
    name: general
  limits:
    cpu: 1000
    memory: 1000Gi
  requirements:
  - key: karpenter.k8s.aws/instance-category
    operator: In
    values: [c, m, r]
  - key: karpenter.k8s.aws/instance-cpu
    operator: In
    values: ["2", "4", "8", "16"]
  - key: karpenter.k8s.aws/instance-generation
    operator: Gt
    values: ["2"]
  - key: karpenter.k8s.aws/instance-hypervisor
    operator: In
    values: [nitro]
  - key: karpenter.k8s.aws/instance-memory
    operator: In
    values: ["4096", "8192", "16384", "32768"]
  - key: karpenter.k8s.aws/instance-network-bandwidth
    operator: Gt
    values: ["500"]
  - key: karpenter.k8s.aws/instance-pods
    operator: In
    values: ["10", "30", "60", "110"]
  - key: karpenter.k8s.aws/instance-size
    operator: In
    values: [small, medium, large, xlarge, 2xlarge, 4xlarge]
  - key: karpenter.k8s.aws/instance-spot
    operator: In
    values: ["true", "false"]
  - key: karpenter.k8s.aws/instance-zone
    operator: In
    values: ["australiaeast-1", "australiaeast-2", "australiaeast-3"]
  - key: kubernetes.io/arch
    operator: In
    values: [amd64]
  - key: kubernetes.io/os
    operator: In
    values: [linux]
  - key: node.kubernetes.io/instance-type
    operator: In
    values: [c5.large, c5.xlarge, c5.2xlarge, m5.large, m5.xlarge, m5.2xlarge, r5.large, r5.xlarge, r5.2xlarge]
  - key: topology.kubernetes.io/zone
    operator: In
    values: [australiaeast-1, australiaeast-2, australiaeast-3]
  weight: 100
EOF
```

### 3.2 Create NodePool for GPU Workloads

```bash
# Create NodePool for GPU workloads
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: NodePool
metadata:
  name: gpu
spec:
  disruption:
    consolidationPolicy: WhenEmpty
    consolidationTTL: 30s
  nodeClassRef:
    apiVersion: karpenter.k8s.aws/v1beta1
    kind: EC2NodeClass
    name: gpu
  limits:
    cpu: 1000
    memory: 1000Gi
  requirements:
  - key: karpenter.k8s.aws/instance-category
    operator: In
    values: [g]
  - key: karpenter.k8s.aws/instance-cpu
    operator: In
    values: ["4", "8", "16", "32"]
  - key: karpenter.k8s.aws/instance-generation
    operator: Gt
    values: ["3"]
  - key: karpenter.k8s.aws/instance-hypervisor
    operator: In
    values: [nitro]
  - key: karpenter.k8s.aws/instance-memory
    operator: In
    values: ["16384", "32768", "65536", "131072"]
  - key: karpenter.k8s.aws/instance-network-bandwidth
    operator: Gt
    values: ["1000"]
  - key: karpenter.k8s.aws/instance-pods
    operator: In
    values: ["30", "60", "110", "234"]
  - key: karpenter.k8s.aws/instance-size
    operator: In
    values: [xlarge, 2xlarge, 4xlarge, 8xlarge, 12xlarge, 16xlarge, 24xlarge]
  - key: karpenter.k8s.aws/instance-spot
    operator: In
    values: ["true", "false"]
  - key: karpenter.k8s.aws/instance-zone
    operator: In
    values: ["australiaeast-1", "australiaeast-2", "australiaeast-3"]
  - key: kubernetes.io/arch
    operator: In
    values: [amd64]
  - key: kubernetes.io/os
    operator: In
    values: [linux]
  - key: node.kubernetes.io/instance-type
    operator: In
    values: [g4dn.xlarge, g4dn.2xlarge, g4dn.4xlarge, g4dn.8xlarge, g4dn.12xlarge, g4dn.16xlarge, g5.xlarge, g5.2xlarge, g5.4xlarge, g5.8xlarge, g5.12xlarge, g5.16xlarge, g5.24xlarge]
  - key: topology.kubernetes.io/zone
    operator: In
    values: [australiaeast-1, australiaeast-2, australiaeast-3]
  weight: 50
EOF
```

### 3.3 Create NodePool for Spot Instances

```bash
# Create NodePool for cost-optimized spot instances
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: NodePool
metadata:
  name: spot
spec:
  disruption:
    consolidationPolicy: WhenEmpty
    consolidationTTL: 30s
  nodeClassRef:
    apiVersion: karpenter.k8s.aws/v1beta1
    kind: EC2NodeClass
    name: spot
  limits:
    cpu: 1000
    memory: 1000Gi
  requirements:
  - key: karpenter.k8s.aws/instance-category
    operator: In
    values: [c, m, r]
  - key: karpenter.k8s.aws/instance-cpu
    operator: In
    values: ["2", "4", "8", "16"]
  - key: karpenter.k8s.aws/instance-generation
    operator: Gt
    values: ["2"]
  - key: karpenter.k8s.aws/instance-hypervisor
    operator: In
    values: [nitro]
  - key: karpenter.k8s.aws/instance-memory
    operator: In
    values: ["4096", "8192", "16384", "32768"]
  - key: karpenter.k8s.aws/instance-network-bandwidth
    operator: Gt
    values: ["500"]
  - key: karpenter.k8s.aws/instance-pods
    operator: In
    values: ["10", "30", "60", "110"]
  - key: karpenter.k8s.aws/instance-size
    operator: In
    values: [small, medium, large, xlarge, 2xlarge, 4xlarge]
  - key: karpenter.k8s.aws/instance-spot
    operator: In
    values: ["true"]
  - key: karpenter.k8s.aws/instance-zone
    operator: In
    values: ["australiaeast-1", "australiaeast-2", "australiaeast-3"]
  - key: kubernetes.io/arch
    operator: In
    values: [amd64]
  - key: kubernetes.io/os
    operator: In
    values: [linux]
  - key: node.kubernetes.io/instance-type
    operator: In
    values: [c5.large, c5.xlarge, c5.2xlarge, m5.large, m5.xlarge, m5.2xlarge, r5.large, r5.xlarge, r5.2xlarge]
  - key: topology.kubernetes.io/zone
    operator: In
    values: [australiaeast-1, australiaeast-2, australiaeast-3]
  weight: 200
EOF
```

---

## Part 4: Creating Provisioners

### 4.1 Create General Workload Provisioner

```bash
# Create Provisioner for general workloads
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: general
spec:
  requirements:
  - key: karpenter.k8s.aws/instance-category
    operator: In
    values: [c, m, r]
  - key: karpenter.k8s.aws/instance-cpu
    operator: In
    values: ["2", "4", "8", "16"]
  - key: karpenter.k8s.aws/instance-generation
    operator: Gt
    values: ["2"]
  - key: karpenter.k8s.aws/instance-hypervisor
    operator: In
    values: [nitro]
  - key: karpenter.k8s.aws/instance-memory
    operator: In
    values: ["4096", "8192", "16384", "32768"]
  - key: karpenter.k8s.aws/instance-network-bandwidth
    operator: Gt
    values: ["500"]
  - key: karpenter.k8s.aws/instance-pods
    operator: In
    values: ["10", "30", "60", "110"]
  - key: karpenter.k8s.aws/instance-size
    operator: In
    values: [small, medium, large, xlarge, 2xlarge, 4xlarge]
  - key: karpenter.k8s.aws/instance-spot
    operator: In
    values: ["true", "false"]
  - key: karpenter.k8s.aws/instance-zone
    operator: In
    values: ["australiaeast-1", "australiaeast-2", "australiaeast-3"]
  - key: kubernetes.io/arch
    operator: In
    values: [amd64]
  - key: kubernetes.io/os
    operator: In
    values: [linux]
  - key: node.kubernetes.io/instance-type
    operator: In
    values: [c5.large, c5.xlarge, c5.2xlarge, m5.large, m5.xlarge, m5.2xlarge, r5.large, r5.xlarge, r5.2xlarge]
  - key: topology.kubernetes.io/zone
    operator: In
    values: [australiaeast-1, australiaeast-2, australiaeast-3]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  ttlSecondsAfterEmpty: 300
  ttlSecondsUntilExpired: 2592000
  weight: 100
  labels:
    workload-type: general
  taints: []
  startupTaints: []
  kubeletConfiguration:
    clusterDNS: ["10.0.0.10"]
    containerRuntime: containerd
    maxPods: 110
    systemReserved:
      cpu: 100m
      memory: 100Mi
      ephemeral-storage: 1Gi
    kubeReserved:
      cpu: 100m
      memory: 100Mi
      ephemeral-storage: 1Gi
    evictionHard:
      memory.available: 100Mi
      nodefs.available: 1%
      nodefs.inodesFree: 1%
    evictionSoft:
      memory.available: 200Mi
      nodefs.available: 1.5%
      nodefs.inodesFree: 1.5%
    evictionSoftGracePeriod:
      memory.available: 2m
      nodefs.available: 2m
      nodefs.inodesFree: 2m
    evictionMaxPodGracePeriod: 60
    imageGCHighThresholdPercent: 85
    imageGCLowThresholdPercent: 80
    cpuCFSQuota: true
    cpuCFSQuotaPeriod: 100us
    cpuManagerPolicy: static
    topologyManagerPolicy: single-numa-node
    logging:
      format: text
      verbosity: 2
    shutdownGracePeriod: 30s
    shutdownGracePeriodCriticalPods: 10s
    memorySwap: {}
    allowedUnsafeSysctls: []
    forbiddenSysctls: []
    clusterDomain: cluster.local
    failSwapOn: true
    generateResolvConf: true
    loggingVerbosity: 0
    readOnlyPort: 10255
    registryBurst: 10
    registryPullQPS: 5
    resolvConf: /etc/resolv.conf
    rotateCertificates: false
    runtimeRequestTimeout: 2m
    serializeImagePulls: true
    streamingConnectionIdleTimeout: 4h
    syncFrequency: 1m
    volumeStatsAggPeriod: 1m
EOF
```

### 4.2 Create GPU Workload Provisioner

```bash
# Create Provisioner for GPU workloads
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: gpu
spec:
  requirements:
  - key: karpenter.k8s.aws/instance-category
    operator: In
    values: [g]
  - key: karpenter.k8s.aws/instance-cpu
    operator: In
    values: ["4", "8", "16", "32"]
  - key: karpenter.k8s.aws/instance-generation
    operator: Gt
    values: ["3"]
  - key: karpenter.k8s.aws/instance-hypervisor
    operator: In
    values: [nitro]
  - key: karpenter.k8s.aws/instance-memory
    operator: In
    values: ["16384", "32768", "65536", "131072"]
  - key: karpenter.k8s.aws/instance-network-bandwidth
    operator: Gt
    values: ["1000"]
  - key: karpenter.k8s.aws/instance-pods
    operator: In
    values: ["30", "60", "110", "234"]
  - key: karpenter.k8s.aws/instance-size
    operator: In
    values: [xlarge, 2xlarge, 4xlarge, 8xlarge, 12xlarge, 16xlarge, 24xlarge]
  - key: karpenter.k8s.aws/instance-spot
    operator: In
    values: ["true", "false"]
  - key: karpenter.k8s.aws/instance-zone
    operator: In
    values: ["australiaeast-1", "australiaeast-2", "australiaeast-3"]
  - key: kubernetes.io/arch
    operator: In
    values: [amd64]
  - key: kubernetes.io/os
    operator: In
    values: [linux]
  - key: node.kubernetes.io/instance-type
    operator: In
    values: [g4dn.xlarge, g4dn.2xlarge, g4dn.4xlarge, g4dn.8xlarge, g4dn.12xlarge, g4dn.16xlarge, g5.xlarge, g5.2xlarge, g5.4xlarge, g5.8xlarge, g5.12xlarge, g5.16xlarge, g5.24xlarge]
  - key: topology.kubernetes.io/zone
    operator: In
    values: [australiaeast-1, australiaeast-2, australiaeast-3]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  ttlSecondsAfterEmpty: 300
  ttlSecondsUntilExpired: 2592000
  weight: 50
  labels:
    workload-type: gpu
    accelerator: nvidia
  taints:
  - key: nvidia.com/gpu
    value: present
    effect: NoSchedule
  startupTaints:
  - key: nvidia.com/gpu
    value: present
    effect: NoSchedule
  kubeletConfiguration:
    clusterDNS: ["10.0.0.10"]
    containerRuntime: containerd
    maxPods: 110
    systemReserved:
      cpu: 100m
      memory: 100Mi
      ephemeral-storage: 1Gi
    kubeReserved:
      cpu: 100m
      memory: 100Mi
      ephemeral-storage: 1Gi
    evictionHard:
      memory.available: 100Mi
      nodefs.available: 1%
      nodefs.inodesFree: 1%
    evictionSoft:
      memory.available: 200Mi
      nodefs.available: 1.5%
      nodefs.inodesFree: 1.5%
    evictionSoftGracePeriod:
      memory.available: 2m
      nodefs.available: 2m
      nodefs.inodesFree: 2m
    evictionMaxPodGracePeriod: 60
    imageGCHighThresholdPercent: 85
    imageGCLowThresholdPercent: 80
    cpuCFSQuota: true
    cpuCFSQuotaPeriod: 100us
    cpuManagerPolicy: static
    topologyManagerPolicy: single-numa-node
    logging:
      format: text
      verbosity: 2
    shutdownGracePeriod: 30s
    shutdownGracePeriodCriticalPods: 10s
    memorySwap: {}
    allowedUnsafeSysctls: []
    forbiddenSysctls: []
    clusterDomain: cluster.local
    failSwapOn: true
    generateResolvConf: true
    loggingVerbosity: 0
    readOnlyPort: 10255
    registryBurst: 10
    registryPullQPS: 5
    resolvConf: /etc/resolv.conf
    rotateCertificates: false
    runtimeRequestTimeout: 2m
    serializeImagePulls: true
    streamingConnectionIdleTimeout: 4h
    syncFrequency: 1m
    volumeStatsAggPeriod: 1m
EOF
```

---

## Part 5: Testing Dynamic Node Provisioning

### 5.1 Create Test Workloads

```bash
# Create namespace for testing
kubectl create namespace karpenter-test

# Create a workload that requires more resources than available
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-intensive-app
  namespace: karpenter-test
spec:
  replicas: 10
  selector:
    matchLabels:
      app: resource-intensive-app
  template:
    metadata:
      labels:
        app: resource-intensive-app
    spec:
      containers:
      - name: app
        image: nginx:alpine
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
        ports:
        - containerPort: 80
EOF

# Create a GPU workload
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-workload
  namespace: karpenter-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gpu-workload
  template:
    metadata:
      labels:
        app: gpu-workload
    spec:
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      containers:
      - name: gpu-app
        image: nvidia/cuda:11.0-base
        command: ["sleep", "infinity"]
        resources:
          requests:
            nvidia.com/gpu: 1
            cpu: "2"
            memory: "8Gi"
          limits:
            nvidia.com/gpu: 1
            cpu: "4"
            memory: "16Gi"
EOF
```

### 5.2 Monitor Node Provisioning

```bash
# Watch Karpenter logs
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller

# Monitor node creation
kubectl get nodes --watch

# Check Karpenter metrics
kubectl port-forward -n karpenter svc/karpenter 8080:8080

# In another terminal, check metrics
curl http://localhost:8080/metrics | grep karpenter

# Monitor pod scheduling
kubectl get pods -n karpenter-test --watch

# Check node pool status
kubectl get nodepools
kubectl get provisioners
```

**Windows PowerShell:**
```powershell
# Watch Karpenter logs
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller

# Monitor node creation
kubectl get nodes --watch

# Check Karpenter metrics
kubectl port-forward -n karpenter svc/karpenter 8080:8080

# In another terminal, check metrics
Invoke-WebRequest -Uri "http://localhost:8080/metrics" | Select-String "karpenter"

# Monitor pod scheduling
kubectl get pods -n karpenter-test --watch

# Check node pool status
kubectl get nodepools
kubectl get provisioners
```

---

## Part 6: Cost Optimization Strategies

### 6.1 Spot Instance Configuration

```bash
# Create a provisioner optimized for spot instances
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: spot-optimized
spec:
  requirements:
  - key: karpenter.k8s.aws/instance-spot
    operator: In
    values: ["true"]
  - key: karpenter.k8s.aws/instance-category
    operator: In
    values: [c, m, r]
  - key: karpenter.k8s.aws/instance-cpu
    operator: In
    values: ["2", "4", "8"]
  - key: karpenter.k8s.aws/instance-memory
    operator: In
    values: ["4096", "8192", "16384"]
  - key: kubernetes.io/arch
    operator: In
    values: [amd64]
  - key: kubernetes.io/os
    operator: In
    values: [linux]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  ttlSecondsAfterEmpty: 300
  ttlSecondsUntilExpired: 2592000
  weight: 300
  labels:
    workload-type: spot
    cost-optimized: "true"
  taints: []
  startupTaints: []
EOF
```

### 6.2 Multi-zone Distribution

```bash
# Create a provisioner for multi-zone distribution
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: multi-zone
spec:
  requirements:
  - key: topology.kubernetes.io/zone
    operator: In
    values: [australiaeast-1, australiaeast-2, australiaeast-3]
  - key: karpenter.k8s.aws/instance-category
    operator: In
    values: [c, m, r]
  - key: karpenter.k8s.aws/instance-cpu
    operator: In
    values: ["2", "4", "8", "16"]
  - key: karpenter.k8s.aws/instance-memory
    operator: In
    values: ["4096", "8192", "16384", "32768"]
  - key: kubernetes.io/arch
    operator: In
    values: [amd64]
  - key: kubernetes.io/os
    operator: In
    values: [linux]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  ttlSecondsAfterEmpty: 300
  ttlSecondsUntilExpired: 2592000
  weight: 100
  labels:
    workload-type: multi-zone
    high-availability: "true"
  taints: []
  startupTaints: []
EOF
```

### 6.3 Resource Bin-packing

```bash
# Create a workload that demonstrates bin-packing
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bin-packing-demo
  namespace: karpenter-test
spec:
  replicas: 20
  selector:
    matchLabels:
      app: bin-packing-demo
  template:
    metadata:
      labels:
        app: bin-packing-demo
    spec:
      containers:
      - name: app
        image: nginx:alpine
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
        ports:
        - containerPort: 80
EOF
```

---

## Part 7: Monitoring and Observability

### 7.1 Set Up Karpenter Dashboard

```bash
# Install Grafana dashboard for Karpenter
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: karpenter-dashboard
  namespace: karpenter
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "Karpenter Dashboard",
        "panels": [
          {
            "title": "Nodes Created",
            "type": "stat",
            "targets": [
              {
                "expr": "karpenter_nodes_created_total"
              }
            ]
          },
          {
            "title": "Nodes Terminated",
            "type": "stat",
            "targets": [
              {
                "expr": "karpenter_nodes_terminated_total"
              }
            ]
          },
          {
            "title": "Provisioning Duration",
            "type": "graph",
            "targets": [
              {
                "expr": "histogram_quantile(0.95, karpenter_provisioner_duration_seconds_bucket)"
              }
            ]
          }
        ]
      }
    }
EOF
```

### 7.2 Create Monitoring Alerts

```bash
# Create alerts for Karpenter
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1alpha1
kind: PrometheusRule
metadata:
  name: karpenter-alerts
  namespace: karpenter
spec:
  groups:
  - name: karpenter
    rules:
    - alert: KarpenterProvisioningFailure
      expr: rate(karpenter_provisioner_failures_total[5m]) > 0
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Karpenter provisioning failures detected"
        description: "Karpenter has failed to provision nodes {{ $value }} times in the last 5 minutes"
    
    - alert: KarpenterHighProvisioningTime
      expr: histogram_quantile(0.95, karpenter_provisioner_duration_seconds_bucket) > 300
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Karpenter provisioning taking too long"
        description: "95th percentile of node provisioning time is {{ $value }} seconds"
    
    - alert: KarpenterNoNodesAvailable
      expr: karpenter_nodes_available == 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "No Karpenter nodes available"
        description: "All Karpenter nodes are unavailable or terminated"
EOF
```

---

## Part 8: Troubleshooting and Best Practices

### 8.1 Common Issues and Solutions

**Node Provisioning Failures:**
```bash
# Check Karpenter logs
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter -c controller --tail=100

# Check node pool status
kubectl describe nodepool general
kubectl describe provisioner general

# Check pod events
kubectl get events -n karpenter-test --sort-by='.lastTimestamp'

# Verify Azure permissions
az role assignment list --assignee $IDENTITY_CLIENT_ID
```

**Performance Issues:**
```bash
# Check Karpenter metrics
kubectl port-forward -n karpenter svc/karpenter 8080:8080
curl http://localhost:8080/metrics | grep karpenter_provisioner

# Monitor resource usage
kubectl top nodes
kubectl top pods -n karpenter

# Check node conditions
kubectl get nodes -o wide
kubectl describe node <node-name>
```

### 8.2 Best Practices

1. **Resource Planning:**
   - Set appropriate resource requests and limits
   - Use node selectors and affinity rules
   - Implement proper pod disruption budgets

2. **Cost Optimization:**
   - Use spot instances for non-critical workloads
   - Implement proper TTL settings
   - Monitor and optimize resource utilization

3. **Security:**
   - Use Pod Security Standards
   - Implement network policies
   - Regular security updates

4. **Monitoring:**
   - Set up comprehensive monitoring
   - Create alerts for critical events
   - Regular performance reviews

---

## Challenges and Exercises

### Challenge 1: Multi-Environment Setup
Create different NodePools and Provisioners for development, staging, and production environments.

### Challenge 2: Cost Optimization
Implement a cost-optimized setup using spot instances and proper resource allocation.

### Challenge 3: High Availability
Set up a multi-zone configuration with proper failover mechanisms.

### Challenge 4: Custom Workloads
Create specialized NodePools for specific workloads (e.g., memory-intensive, compute-intensive).

---

## Cleanup

```bash
# Delete test workloads
kubectl delete namespace karpenter-test

# Delete NodePools and Provisioners
kubectl delete nodepool general gpu spot
kubectl delete provisioner general gpu spot-optimized multi-zone

# Uninstall Karpenter
helm uninstall karpenter -n karpenter
kubectl delete namespace karpenter

# Remove managed identity
az identity delete \
  --resource-group "rg-aks-workshop-01" \
  --name "karpenter-identity"

# Disable NAP (optional)
az aks update \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --disable-node-autoprovisioning
```

**Windows PowerShell:**
```powershell
# Delete test workloads
kubectl delete namespace karpenter-test

# Delete NodePools and Provisioners
kubectl delete nodepool general gpu spot
kubectl delete provisioner general gpu spot-optimized multi-zone

# Uninstall Karpenter
helm uninstall karpenter -n karpenter
kubectl delete namespace karpenter

# Remove managed identity
az identity delete `
  --resource-group "rg-aks-workshop-01" `
  --name "karpenter-identity"

# Disable NAP (optional)
az aks update `
  --resource-group "rg-aks-workshop-01" `
  --name "aks-workshop-01-cluster" `
  --disable-node-autoprovisioning
```

---

## Summary

In this exercise, you learned:

✅ **Node Auto Provisioner Concepts**: Understanding NAP and Karpenter architecture  
✅ **Karpenter Installation**: Setting up Karpenter in AKS with proper authentication  
✅ **NodePool Configuration**: Creating different NodePools for various workload types  
✅ **Provisioner Setup**: Configuring Provisioners with specific requirements  
✅ **Dynamic Scaling**: Testing automatic node provisioning and scaling  
✅ **Cost Optimization**: Implementing spot instances and resource bin-packing  
✅ **Monitoring**: Setting up observability and alerting for Karpenter  
✅ **Troubleshooting**: Common issues and best practices for NAP  

## Next Steps

- Implement NAP in production environments
- Set up advanced cost optimization strategies
- Configure multi-cluster NAP setups
- Integrate with existing monitoring and alerting systems
- Develop custom NodePools for specific business requirements

## Additional Resources

- [Karpenter Documentation](https://karpenter.sh/)
- [AKS Node Auto Provisioner](https://docs.microsoft.com/en-us/azure/aks/cluster-autoscaler)
- [Azure Spot Instances](https://docs.microsoft.com/en-us/azure/virtual-machines/spot-vms)
- [Kubernetes Node Management](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [Karpenter Best Practices](https://karpenter.sh/docs/concepts/best-practices/)

---

**Congratulations!** You've completed the advanced Node Auto Provisioner exercise. You now have comprehensive knowledge of dynamic node provisioning and cost optimization in AKS. 