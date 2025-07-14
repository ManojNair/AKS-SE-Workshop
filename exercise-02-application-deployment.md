# Exercise 2: Application Deployment and Management

## Overview

In this exercise, you will learn how to deploy and manage applications in your AKS cluster. You'll work with Kubernetes manifests, different service types, and understand how networking works in AKS.

## Learning Objectives

By the end of this exercise, you will be able to:
- Deploy applications using kubectl and YAML manifests
- Understand different Kubernetes resource types (Pods, Deployments, Services)
- Work with different service types (ClusterIP, LoadBalancer, NodePort)
- Manage application updates and rollbacks
- Configure environment variables and secrets
- Use ConfigMaps for configuration management

## Prerequisites

- Completed Exercise 1: AKS Cluster Fundamentals
- AKS cluster running and accessible via kubectl
- Basic understanding of YAML syntax

## Exercise Steps

### Step 1: Prepare Your Environment

First, ensure you're connected to your AKS cluster:

```bash
# Verify cluster connection
kubectl cluster-info

# Check current context
kubectl config current-context

# List nodes to ensure cluster is healthy
kubectl get nodes
```

### Step 2: Create a Namespace for Your Applications

Create a dedicated namespace for this exercise:

```bash
# Create namespace
kubectl create namespace app-workshop

# Verify namespace creation
kubectl get namespaces

# Set the namespace as default for this session
kubectl config set-context --current --namespace=app-workshop
```

### Step 3: Deploy Your First Application

Start with a simple nginx deployment:

```bash
# Deploy nginx using kubectl create deployment
kubectl create deployment nginx-app \
  --image=nginx:latest \
  --replicas=2 \
  --namespace=app-workshop

# Check the deployment
kubectl get deployments --namespace=app-workshop

# Check the pods
kubectl get pods --namespace=app-workshop

# Get detailed information about the deployment
kubectl describe deployment nginx-app --namespace=app-workshop
```

### Step 4: Create a Service to Expose Your Application

Create a LoadBalancer service to make your application accessible:

```bash
# Create LoadBalancer service
kubectl expose deployment nginx-app \
  --type=LoadBalancer \
  --port=80 \
  --target-port=80 \
  --namespace=app-workshop

# Check the service
kubectl get services --namespace=app-workshop

# Get the external IP (this may take a few minutes)
kubectl get service nginx-app --namespace=app-workshop -w
```

### Step 5: Work with YAML Manifests

Create a more complex application using YAML manifests. Create a file called `web-app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: app-workshop
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: nginx:1.21
        ports:
        - containerPort: 80
        env:
        - name: NGINX_HOST
          value: "localhost"
        - name: NGINX_PORT
          value: "80"
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
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
  name: web-app-service
  namespace: app-workshop
  labels:
    app: web-app
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  selector:
    app: web-app
```

Deploy the application using the YAML file:

```bash
# Apply the YAML manifest
kubectl apply -f web-app.yaml

# Check the deployment
kubectl get deployments --namespace=app-workshop

# Check the service
kubectl get services --namespace=app-workshop

# Check the pods
kubectl get pods --namespace=app-workshop
```

### Step 6: Create a ConfigMap for Configuration

Create a ConfigMap to store configuration data:

```bash
# Create a ConfigMap
kubectl create configmap app-config \
  --from-literal=APP_ENV=development \
  --from-literal=LOG_LEVEL=info \
  --from-literal=API_URL=https://api.example.com \
  --namespace=app-workshop

# Verify the ConfigMap
kubectl get configmaps --namespace=app-workshop

# Describe the ConfigMap
kubectl describe configmap app-config --namespace=app-workshop
```

### Step 7: Create a Secret for Sensitive Data

Create a Secret to store sensitive information:

```bash
# Create a Secret
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD=mysecretpassword \
  --from-literal=API_KEY=myapikey123 \
  --namespace=app-workshop

# Verify the Secret
kubectl get secrets --namespace=app-workshop

# Describe the Secret (note: values are base64 encoded)
kubectl describe secret app-secret --namespace=app-workshop
```

### Step 8: Update Your Application to Use ConfigMap and Secret

Create an updated deployment file called `web-app-with-config.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-with-config
  namespace: app-workshop
  labels:
    app: web-app-with-config
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app-with-config
  template:
    metadata:
      labels:
        app: web-app-with-config
    spec:
      containers:
      - name: web-app
        image: nginx:1.21
        ports:
        - containerPort: 80
        env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
        - name: API_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: API_URL
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: API_KEY
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

Deploy the updated application:

```bash
# Apply the updated deployment
kubectl apply -f web-app-with-config.yaml

# Check the new deployment
kubectl get deployments --namespace=app-workshop

# Check the pods
kubectl get pods --namespace=app-workshop

# Verify environment variables in a pod
kubectl exec -it $(kubectl get pods --namespace=app-workshop -l app=web-app-with-config -o jsonpath='{.items[0].metadata.name}') --namespace=app-workshop -- env | grep -E "(APP_ENV|LOG_LEVEL|API_URL|DB_PASSWORD|API_KEY)"
```

### Step 9: Perform Application Updates and Rollbacks

Update your application to a newer version:

```bash
# Update the image to a newer version
kubectl set image deployment/web-app-with-config web-app=nginx:1.22 --namespace=app-workshop

# Check the rollout status
kubectl rollout status deployment/web-app-with-config --namespace=app-workshop

# Check the deployment history
kubectl rollout history deployment/web-app-with-config --namespace=app-workshop

# Simulate a problematic update
kubectl set image deployment/web-app-with-config web-app=nginx:invalid --namespace=app-workshop

# Check the rollout status (this will fail)
kubectl rollout status deployment/web-app-with-config --namespace=app-workshop

# Rollback to the previous version
kubectl rollout undo deployment/web-app-with-config --namespace=app-workshop

# Verify the rollback
kubectl rollout status deployment/web-app-with-config --namespace=app-workshop
```

### Step 10: Explore Different Service Types

Create different types of services to understand their use cases:

#### ClusterIP Service (Internal Access)
```bash
# Create a ClusterIP service
kubectl expose deployment web-app-with-config \
  --type=ClusterIP \
  --port=80 \
  --target-port=80 \
  --name=web-app-clusterip \
  --namespace=app-workshop

# Check the service
kubectl get service web-app-clusterip --namespace=app-workshop
```

#### NodePort Service (External Access via Node Port)
```bash
# Create a NodePort service
kubectl expose deployment web-app-with-config \
  --type=NodePort \
  --port=80 \
  --target-port=80 \
  --name=web-app-nodeport \
  --namespace=app-workshop

# Check the service
kubectl get service web-app-nodeport --namespace=app-workshop
```

### Step 11: Test Application Connectivity

Test different ways to access your application:

```bash
# Get the LoadBalancer external IP
EXTERNAL_IP=$(kubectl get service web-app-service --namespace=app-workshop -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Test the application (if you have curl installed)
curl -I http://$EXTERNAL_IP

# Or use kubectl port-forward for testing
kubectl port-forward service/web-app-service 8080:80 --namespace=app-workshop

# In another terminal, test locally
curl -I http://localhost:8080
```

## Understanding Kubernetes Resources

### Key Resource Types

1. **Pods**
   - Smallest deployable units in Kubernetes
   - Contain one or more containers
   - Ephemeral by nature

2. **Deployments**
   - Manage the desired state for Pods and ReplicaSets
   - Provide declarative updates for Pods and ReplicaSets
   - Support rollbacks and scaling

3. **Services**
   - Abstract way to expose applications running on Pods
   - Provide load balancing and service discovery
   - Different types: ClusterIP, NodePort, LoadBalancer

4. **ConfigMaps**
   - Store non-confidential configuration data
   - Can be consumed by Pods as environment variables or files
   - Useful for application configuration

5. **Secrets**
   - Store sensitive data like passwords, tokens, keys
   - Base64 encoded by default
   - Can be consumed by Pods as environment variables or files

### Exercise Questions

Answer these questions to reinforce your learning:

1. **What is the difference between a Pod and a Deployment?**
   - Pods are the smallest units, while Deployments manage Pod lifecycle
   - Deployments provide scaling, updates, and rollback capabilities
   - Pods are ephemeral, Deployments ensure desired state

2. **When would you use ClusterIP vs LoadBalancer service?**
   - ClusterIP: Internal communication between services
   - LoadBalancer: External access from outside the cluster

3. **What is the purpose of ConfigMaps and Secrets?**
   - ConfigMaps: Store non-sensitive configuration data
   - Secrets: Store sensitive data like passwords and API keys

## Hands-on Challenges

### Challenge 1: Multi-Tier Application
Deploy a multi-tier application with frontend and backend:

```yaml
# Create frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: app-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: app-workshop
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: frontend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: app-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: app-workshop
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: backend
```

### Challenge 2: Application Scaling
Practice scaling your applications:

```bash
# Scale the frontend deployment
kubectl scale deployment frontend --replicas=5 --namespace=app-workshop

# Check the scaling
kubectl get pods --namespace=app-workshop

# Scale back down
kubectl scale deployment frontend --replicas=2 --namespace=app-workshop
```

### Challenge 3: Health Checks
Add health checks to your application:

```yaml
# Add to your deployment spec
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
```

## Troubleshooting

### Common Issues

1. **Pod Stuck in Pending State**
   ```bash
   # Check pod events
   kubectl describe pod <pod-name> --namespace=app-workshop
   
   # Check node resources
   kubectl top nodes
   ```

2. **Service Not Accessible**
   ```bash
   # Check service endpoints
   kubectl get endpoints --namespace=app-workshop
   
   # Check service configuration
   kubectl describe service <service-name> --namespace=app-workshop
   ```

3. **Application Not Starting**
   ```bash
   # Check pod logs
   kubectl logs <pod-name> --namespace=app-workshop
   
   # Check pod status
   kubectl describe pod <pod-name> --namespace=app-workshop
   ```

## Cleanup

Clean up resources from this exercise:

```bash
# Delete all resources in the namespace
kubectl delete namespace app-workshop

# Or delete individual resources
kubectl delete deployment web-app --namespace=app-workshop
kubectl delete service web-app-service --namespace=app-workshop
kubectl delete configmap app-config --namespace=app-workshop
kubectl delete secret app-secret --namespace=app-workshop
```

## Summary

In this exercise, you have:
- ✅ Deployed applications using kubectl and YAML manifests
- ✅ Worked with different Kubernetes resource types
- ✅ Created and used ConfigMaps and Secrets
- ✅ Performed application updates and rollbacks
- ✅ Explored different service types
- ✅ Tested application connectivity

## Next Steps

You're now ready to move to **Exercise 3: Scaling and Resource Management** where you'll learn about horizontal and vertical scaling, resource quotas, and cluster autoscaler.

## Additional Resources

- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) 