# Exercise 3: Scaling and Resource Management

## Overview

In this exercise, you will learn about scaling strategies and resource management in AKS. You'll explore horizontal and vertical scaling, resource quotas, limits, and the cluster autoscaler.

## Learning Objectives

By the end of this exercise, you will be able to:
- Implement horizontal and vertical scaling strategies
- Configure resource requests and limits
- Set up resource quotas and limits
- Enable and configure cluster autoscaler
- Monitor resource usage and performance
- Optimize resource allocation for cost efficiency

## Prerequisites

- Completed Exercise 2: Application Deployment and Management
- AKS cluster running with sufficient resources
- Understanding of basic Kubernetes concepts

## Exercise Steps

### Step 1: Prepare Your Environment

First, ensure you're connected to your AKS cluster and create a namespace for this exercise:

```bash
# Verify cluster connection
kubectl cluster-info

# Create namespace for this exercise
kubectl create namespace scaling-workshop

# Set the namespace as default
kubectl config set-context --current --namespace=scaling-workshop

# Check current cluster resources
kubectl top nodes
kubectl top pods --all-namespaces
```

### Step 2: Deploy a Test Application

Create a test application that we'll use for scaling experiments:

```yaml
# Create scaling-test-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scaling-test-app
  namespace: scaling-workshop
  labels:
    app: scaling-test-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: scaling-test-app
  template:
    metadata:
      labels:
        app: scaling-test-app
    spec:
      containers:
      - name: scaling-test-app
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
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: scaling-test-service
  namespace: scaling-workshop
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: scaling-test-app
```

Deploy the application:

```bash
# Apply the deployment
kubectl apply -f scaling-test-app.yaml

# Check the deployment
kubectl get deployments --namespace=scaling-workshop

# Check the pods
kubectl get pods --namespace=scaling-workshop

# Check resource usage
kubectl top pods --namespace=scaling-workshop
```

### Step 3: Horizontal Pod Scaling (HPA)

Set up Horizontal Pod Autoscaler to automatically scale your application based on CPU usage:

```bash
# Create HPA for CPU-based scaling
kubectl autoscale deployment scaling-test-app \
  --cpu-percent=50 \
  --min=2 \
  --max=10 \
  --namespace=scaling-workshop

# Check HPA status
kubectl get hpa --namespace=scaling-workshop

# Describe HPA for detailed information
kubectl describe hpa scaling-test-app --namespace=scaling-workshop
```

### Step 4: Test Horizontal Scaling

Generate load to test the HPA:

```bash
# Create a load generator pod
kubectl run load-generator \
  --image=busybox \
  --namespace=scaling-workshop \
  --restart=Never \
  --rm -it \
  -- sh -c "while true; do wget -q -O- http://scaling-test-service; sleep 0.1; done"

# In another terminal, monitor the scaling
kubectl get hpa scaling-test-app --watch --namespace=scaling-workshop

# Check pod count
kubectl get pods --watch --namespace=scaling-workshop
```

### Step 5: Manual Horizontal Scaling

Practice manual scaling operations:

```bash
# Scale up manually
kubectl scale deployment scaling-test-app --replicas=5 --namespace=scaling-workshop

# Check the scaling
kubectl get pods --namespace=scaling-workshop

# Scale down
kubectl scale deployment scaling-test-app --replicas=2 --namespace=scaling-workshop

# Check resource usage after scaling
kubectl top pods --namespace=scaling-workshop
```

### Step 6: Resource Requests and Limits

Create an application with different resource configurations to understand their impact:

```yaml
# Create resource-test-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-test-app
  namespace: scaling-workshop
spec:
  replicas: 3
  selector:
    matchLabels:
      app: resource-test-app
  template:
    metadata:
      labels:
        app: resource-test-app
    spec:
      containers:
      - name: resource-test-app
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "200m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        env:
        - name: NGINX_WORKER_PROCESSES
          value: "2"
        - name: NGINX_WORKER_CONNECTIONS
          value: "1024"
```

Deploy and test resource limits:

```bash
# Apply the deployment
kubectl apply -f resource-test-app.yaml

# Check resource allocation
kubectl describe pods --namespace=scaling-workshop | grep -A 5 "Containers:"

# Monitor resource usage
kubectl top pods --namespace=scaling-workshop
```

### Step 7: Resource Quotas

Set up resource quotas to limit resource consumption in the namespace:

```yaml
# Create resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: scaling-workshop-quota
  namespace: scaling-workshop
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
    pods: "10"
    services: "5"
    persistentvolumeclaims: "5"
```

Apply the resource quota:

```bash
# Apply the quota
kubectl apply -f resource-quota.yaml

# Check quota status
kubectl get resourcequota --namespace=scaling-workshop

# Describe quota for detailed information
kubectl describe resourcequota scaling-workshop-quota --namespace=scaling-workshop
```

### Step 8: Test Resource Quotas

Try to exceed the quota limits:

```bash
# Try to create more pods than allowed
kubectl scale deployment scaling-test-app --replicas=15 --namespace=scaling-workshop

# Check the quota status
kubectl get resourcequota scaling-workshop-quota --namespace=scaling-workshop

# Check pod status
kubectl get pods --namespace=scaling-workshop

# Scale back down
kubectl scale deployment scaling-test-app --replicas=3 --namespace=scaling-workshop
```

### Step 9: Cluster Autoscaler

Enable cluster autoscaler to automatically scale your node pool:

```bash
# Enable cluster autoscaler
az aks update \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 5

# Check autoscaler status
az aks show \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --query "agentPoolProfiles[0].enableAutoScaling"
```

### Step 10: Test Cluster Autoscaler

Create a workload that requires more nodes:

```bash
# Create a high-resource deployment
kubectl create deployment high-resource-app \
  --image=nginx:latest \
  --replicas=10 \
  --namespace=scaling-workshop

# Set resource requests that will require more nodes
kubectl patch deployment high-resource-app \
  --namespace=scaling-workshop \
  --patch='{"spec":{"template":{"spec":{"containers":[{"name":"high-resource-app","resources":{"requests":{"cpu":"500m","memory":"512Mi"}}}]}}}}'

# Monitor node scaling
kubectl get nodes --watch

# Check pod scheduling
kubectl get pods --namespace=scaling-workshop
```

### Step 11: Vertical Pod Autoscaler (VPA)

Set up VPA for automatic resource optimization:

```bash
# Install VPA (if not already installed)
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler/

# Apply VPA components
kubectl apply -f hack/vpa-rbac.yaml
kubectl apply -f hack/vpa-rbac.yaml

# Create VPA for your application
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: scaling-test-app-vpa
  namespace: scaling-workshop
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: scaling-test-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: 100m
        memory: 50Mi
      maxAllowed:
        cpu: 1
        memory: 500Mi
      controlledResources: ["cpu", "memory"]
EOF

# Check VPA status
kubectl get vpa --namespace=scaling-workshop
kubectl describe vpa scaling-test-app-vpa --namespace=scaling-workshop
```

### Step 12: Monitor and Analyze

Set up comprehensive monitoring for your scaling experiments:

```bash
# Check current resource usage
kubectl top nodes
kubectl top pods --all-namespaces

# Get detailed resource information
kubectl describe nodes | grep -A 10 "Allocated resources"

# Check HPA metrics
kubectl get hpa --namespace=scaling-workshop -o yaml

# Monitor cluster autoscaler logs
kubectl logs -n kube-system deployment/cluster-autoscaler --tail=50
```

## Understanding Scaling Concepts

### Horizontal vs Vertical Scaling

1. **Horizontal Scaling (Scale Out/In)**
   - Add/remove instances of the same application
   - Better for distributed workloads
   - Easier to implement in Kubernetes
   - Example: Increasing replica count

2. **Vertical Scaling (Scale Up/Down)**
   - Increase/decrease resources per instance
   - Better for single-threaded applications
   - Requires pod restart in Kubernetes
   - Example: Increasing CPU/memory limits

### Resource Management Best Practices

1. **Resource Requests**
   - Always set resource requests
   - Helps with pod scheduling
   - Prevents resource starvation

2. **Resource Limits**
   - Set limits to prevent resource hogging
   - Protects against runaway processes
   - Consider application requirements

3. **Quotas and Limits**
   - Use quotas to prevent resource abuse
   - Set namespace-level limits
   - Monitor quota usage regularly

### Exercise Questions

Answer these questions to reinforce your learning:

1. **What is the difference between HPA and VPA?**
   - HPA scales the number of pods based on metrics
   - VPA adjusts resource requests/limits for pods
   - HPA is more commonly used in production

2. **When should you use cluster autoscaler?**
   - When you have variable workloads
   - To optimize costs during low usage
   - When you want automatic node management

3. **What are the benefits of setting resource requests and limits?**
   - Better pod scheduling decisions
   - Prevents resource starvation
   - Enables proper resource planning

## Hands-on Challenges

### Challenge 1: Multi-Metric HPA
Create an HPA that scales based on both CPU and memory:

```bash
# Create custom metrics server (if needed)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Create HPA with custom metrics
kubectl autoscale deployment scaling-test-app \
  --cpu-percent=50 \
  --min=2 \
  --max=10 \
  --namespace=scaling-workshop
```

### Challenge 2: Custom Metrics Scaling
Set up scaling based on custom application metrics:

```yaml
# Create custom metrics HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: custom-metrics-hpa
  namespace: scaling-workshop
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: scaling-test-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Challenge 3: Cost Optimization
Implement cost optimization strategies:

```bash
# Scale down during off-hours
kubectl scale deployment scaling-test-app --replicas=1 --namespace=scaling-workshop

# Use spot instances for non-critical workloads
az aks nodepool add \
  --resource-group "rg-aks-workshop-01" \
  --cluster-name "aks-workshop-01-cluster" \
  --name spotpool \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 3 \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1
```

## Troubleshooting

### Common Issues

1. **HPA Not Scaling**
   ```bash
   # Check metrics server
   kubectl get pods -n kube-system | grep metrics-server
   
   # Check HPA events
   kubectl describe hpa scaling-test-app --namespace=scaling-workshop
   ```

2. **Cluster Autoscaler Not Working**
   ```bash
   # Check autoscaler logs
   kubectl logs -n kube-system deployment/cluster-autoscaler
   
   # Verify autoscaler configuration
   az aks show \
     --resource-group "rg-aks-workshop-01" \
     --name "aks-workshop-01-cluster" \
     --query "agentPoolProfiles[0].enableAutoScaling"
   ```

3. **Resource Quota Exceeded**
   ```bash
   # Check quota status
   kubectl get resourcequota --namespace=scaling-workshop
   
   # Check pod events
   kubectl get events --namespace=scaling-workshop
   ```

## Cleanup

Clean up resources from this exercise:

```bash
# Delete all resources in the namespace
kubectl delete namespace scaling-workshop

# Disable cluster autoscaler
az aks update \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --disable-cluster-autoscaler

# Scale cluster back to original size
az aks scale \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --node-count 2
```

## Summary

In this exercise, you have:
- ✅ Implemented horizontal and vertical scaling strategies
- ✅ Configured resource requests and limits
- ✅ Set up resource quotas and limits
- ✅ Enabled and configured cluster autoscaler
- ✅ Monitored resource usage and performance
- ✅ Optimized resource allocation for cost efficiency

## Next Steps

You're now ready to move to **Exercise 4: Networking and Security** where you'll learn about network policies, Azure CNI, RBAC, and security best practices.

## Additional Resources

- [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubernetes VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [AKS Cluster Autoscaler](https://docs.microsoft.com/en-us/azure/aks/cluster-autoscaler)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [AKS Best Practices](https://docs.microsoft.com/en-us/azure/aks/best-practices) 