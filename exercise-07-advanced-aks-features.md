# Exercise 7: Advanced AKS Features

## Overview

In this exercise, you will explore advanced AKS features including virtual nodes, GPU workloads, Windows containers, and other enterprise-grade capabilities.

## Learning Objectives

- Set up and use virtual nodes (serverless)
- Deploy GPU-enabled workloads
- Work with Windows containers
- Configure advanced networking features
- Use Azure CNI with custom subnet delegation
- Implement advanced security features

## Prerequisites

- Completed Exercise 6: CI/CD with AKS
- AKS cluster with sufficient resources
- Understanding of basic AKS concepts

## Exercise Steps

### Step 1: Prepare Your Environment

```bash
# Verify cluster connection
kubectl cluster-info

# Create namespace for this exercise
kubectl create namespace advanced-workshop

# Set the namespace as default
kubectl config set-context --current --namespace=advanced-workshop

# Check current cluster configuration
az aks show \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --query "{kubernetesVersion:kubernetesVersion, networkProfile:networkProfile, addonProfiles:addonProfiles}"
```

### Step 2: Set Up Virtual Nodes (Serverless)

Enable virtual nodes for serverless workloads:

```bash
# Enable virtual nodes add-on
az aks enable-addons \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --addons virtual-node \
  --subnet-name aks-subnet

# Check virtual nodes
kubectl get nodes -o wide

# Check virtual node pods
kubectl get pods -n kube-system | grep virtual-node
```

### Step 3: Deploy Serverless Workloads

Create workloads that use virtual nodes:

```yaml
# Create serverless-workload.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: serverless-app
  namespace: advanced-workshop
spec:
  replicas: 5
  selector:
    matchLabels:
      app: serverless-app
  template:
    metadata:
      labels:
        app: serverless-app
    spec:
      nodeSelector:
        kubernetes.azure.com/mode: virtual
      tolerations:
      - key: "kubernetes.azure.com/scalesetpriority"
        operator: "Equal"
        value: "spot"
        effect: "NoSchedule"
      containers:
      - name: serverless-app
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: serverless-service
  namespace: advanced-workshop
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: serverless-app
```

Deploy the serverless workload:

```bash
# Apply serverless workload
kubectl apply -f serverless-workload.yaml

# Check deployment
kubectl get pods -o wide

# Check which nodes the pods are running on
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
```

### Step 4: Set Up GPU Node Pool

Create a GPU-enabled node pool for compute-intensive workloads:

```bash
# Create GPU node pool
az aks nodepool add \
  --resource-group "rg-aks-workshop-01" \
  --cluster-name "aks-workshop-01-cluster" \
  --name gpunodepool \
  --node-count 1 \
  --node-vm-size Standard_NC6s_v3 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 2 \
  --node-taints sku=gpu:NoSchedule

# Check node pool creation
az aks nodepool list \
  --resource-group "rg-aks-workshop-01" \
  --cluster-name "aks-workshop-01-cluster"

# Check nodes
kubectl get nodes --show-labels | grep gpu
```

### Step 5: Deploy GPU Workloads

Create GPU-enabled workloads:

```yaml
# Create gpu-workload.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-app
  namespace: advanced-workshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gpu-app
  template:
    metadata:
      labels:
        app: gpu-app
    spec:
      tolerations:
      - key: "sku"
        operator: "Equal"
        value: "gpu"
        effect: "NoSchedule"
      containers:
      - name: gpu-app
        image: nvidia/cuda:11.0-base-ubuntu20.04
        command: ["/bin/bash"]
        args: ["-c", "nvidia-smi && sleep 3600"]
        resources:
          limits:
            nvidia.com/gpu: 1
          requests:
            nvidia.com/gpu: 1
---
apiVersion: v1
kind: Service
metadata:
  name: gpu-app-service
  namespace: advanced-workshop
spec:
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: gpu-app
```

Deploy the GPU workload:

```bash
# Apply GPU workload
kubectl apply -f gpu-workload.yaml

# Check GPU pods
kubectl get pods -l app=gpu-app

# Check GPU usage
kubectl exec -it $(kubectl get pods -l app=gpu-app -o jsonpath='{.items[0].metadata.name}') -- nvidia-smi
```

### Step 6: Set Up Windows Node Pool

Create a Windows node pool for Windows containers:

```bash
# Create Windows node pool
az aks nodepool add \
  --resource-group "rg-aks-workshop-01" \
  --cluster-name "aks-workshop-01-cluster" \
  --name winodepool \
  --node-count 1 \
  --node-vm-size Standard_D2s_v3 \
  --os-type Windows \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 2

# Check Windows nodes
kubectl get nodes --show-labels | grep windows

# Wait for Windows nodes to be ready
kubectl wait --for=condition=ready node -l kubernetes.io/os=windows --timeout=600s
```

### Step 7: Deploy Windows Containers

Create Windows container workloads:

```yaml
# Create windows-workload.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windows-app
  namespace: advanced-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: windows-app
  template:
    metadata:
      labels:
        app: windows-app
    spec:
      nodeSelector:
        kubernetes.io/os: windows
      containers:
      - name: windows-app
        image: mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2019
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: windows-app-service
  namespace: advanced-workshop
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: windows-app
```

Deploy the Windows workload:

```bash
# Apply Windows workload
kubectl apply -f windows-workload.yaml

# Check Windows pods
kubectl get pods -l app=windows-app

# Wait for Windows pods to be ready
kubectl wait --for=condition=ready pod -l app=windows-app --timeout=600s
```

### Step 8: Configure Advanced Networking

Set up advanced networking features:

```bash
# Get current network configuration
VNET_NAME=$(az aks show \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --query "agentPoolProfiles[0].vnetSubnetId" -o tsv | cut -d'/' -f9)

SUBNET_NAME=$(az aks show \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --query "agentPoolProfiles[0].vnetSubnetId" -o tsv | cut -d'/' -f11)

echo "VNet: $VNET_NAME"
echo "Subnet: $SUBNET_NAME"

# Enable subnet delegation for AKS
az network vnet subnet update \
  --resource-group "rg-aks-workshop-01" \
  --vnet-name "$VNET_NAME" \
  --name "$SUBNET_NAME" \
  --delegations Microsoft.ContainerService/managedClusters
```

### Step 9: Create Advanced Network Policies

Implement sophisticated network policies:

```yaml
# Create advanced-network-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: advanced-policy
  namespace: advanced-workshop
spec:
  podSelector:
    matchLabels:
      app: serverless-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: advanced-workshop
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 80
```

Apply advanced network policies:

```bash
# Apply network policies
kubectl apply -f advanced-network-policies.yaml

# Check network policies
kubectl get networkpolicies --namespace=advanced-workshop
```

### Step 10: Implement Advanced Security Features

Set up advanced security configurations:

```yaml
# Create pod-security-policy.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: advanced-psp
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'runtime/default'
    apparmor.security.beta.kubernetes.io/allowedProfileNames: 'runtime/default'
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  readOnlyRootFilesystem: true
```

### Step 11: Test Advanced Features

Test the advanced features you've implemented:

```bash
# Test virtual nodes
kubectl get pods -l app=serverless-app -o wide

# Test GPU workloads
kubectl exec -it $(kubectl get pods -l app=gpu-app -o jsonpath='{.items[0].metadata.name}') -- nvidia-smi

# Test Windows containers
kubectl get pods -l app=windows-app -o wide

# Test network policies
kubectl exec -it $(kubectl get pods -l app=serverless-app -o jsonpath='{.items[0].metadata.name}') -- wget -q -O- http://google.com

# Check resource usage across different node types
kubectl top nodes
kubectl top pods --all-namespaces
```

### Step 12: Monitor Advanced Features

Set up monitoring for advanced features:

```bash
# Check node pool status
az aks nodepool list \
  --resource-group "rg-aks-workshop-01" \
  --cluster-name "aks-workshop-01-cluster" \
  --output table

# Check virtual node status
kubectl get pods -n kube-system | grep virtual-node

# Check GPU node status
kubectl get nodes --show-labels | grep gpu

# Check Windows node status
kubectl get nodes --show-labels | grep windows

# Monitor resource usage
kubectl top nodes
kubectl top pods --all-namespaces
```

## Understanding Advanced AKS Features

### Virtual Nodes (Serverless)

1. **Benefits**
   - Pay only for what you use
   - Instant scaling
   - No node management

2. **Use Cases**
   - Batch processing
   - Development/testing
   - Variable workloads

### GPU Workloads

1. **GPU Node Pools**
   - Specialized VM sizes
   - NVIDIA drivers pre-installed
   - Cost optimization with autoscaler

2. **Use Cases**
   - Machine learning
   - Data processing
   - Scientific computing

### Windows Containers

1. **Windows Node Pools**
   - Windows Server containers
   - .NET applications
   - Legacy application migration

2. **Considerations**
   - Larger image sizes
   - Different networking behavior
   - Resource requirements

## Hands-on Challenges

### Challenge 1: Multi-Architecture Deployment
Deploy the same application across Linux, Windows, and GPU nodes.

### Challenge 2: Advanced Scaling
Set up different scaling policies for different node pools.

### Challenge 3: Cost Optimization
Implement spot instances and autoscaling for cost optimization.

## Troubleshooting

### Common Issues

1. **Virtual Nodes Not Working**
   ```bash
   # Check virtual node add-on
   az aks show \
     --resource-group "rg-aks-workshop-01" \
     --name "aks-workshop-01-cluster" \
     --query "addonProfiles.virtualNode.enabled"
   
   # Check virtual node pods
   kubectl get pods -n kube-system | grep virtual-node
   ```

2. **GPU Workloads Not Starting**
   ```bash
   # Check GPU nodes
   kubectl get nodes --show-labels | grep gpu
   
   # Check GPU drivers
   kubectl exec -it <gpu-pod> -- nvidia-smi
   ```

3. **Windows Containers Issues**
   ```bash
   # Check Windows nodes
   kubectl get nodes --show-labels | grep windows
   
   # Check Windows pod events
   kubectl describe pod <windows-pod>
   ```

## Cleanup

Clean up resources from this exercise:

```bash
# Delete all resources in the namespace
kubectl delete namespace advanced-workshop

# Delete GPU node pool
az aks nodepool delete \
  --resource-group "rg-aks-workshop-01" \
  --cluster-name "aks-workshop-01-cluster" \
  --name gpunodepool

# Delete Windows node pool
az aks nodepool delete \
  --resource-group "rg-aks-workshop-01" \
  --cluster-name "aks-workshop-01-cluster" \
  --name winodepool

# Disable virtual nodes
az aks disable-addons \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --addons virtual-node
```

## Summary

In this exercise, you have:
- ✅ Set up and used virtual nodes (serverless)
- ✅ Deployed GPU-enabled workloads
- ✅ Worked with Windows containers
- ✅ Configured advanced networking features
- ✅ Used Azure CNI with custom subnet delegation
- ✅ Implemented advanced security features

## Next Steps

You're now ready to move to **Exercise 8: Disaster Recovery and Backup** where you'll learn about multi-region deployment and business continuity.

## Additional Resources

- [AKS Virtual Nodes](https://docs.microsoft.com/en-us/azure/aks/virtual-nodes)
- [GPU Workloads on AKS](https://docs.microsoft.com/en-us/azure/aks/gpu-cluster)
- [Windows Containers on AKS](https://docs.microsoft.com/en-us/azure/aks/windows-container-cli)
- [AKS Advanced Networking](https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni)
- [AKS Security Best Practices](https://docs.microsoft.com/en-us/azure/aks/security-best-practices) 