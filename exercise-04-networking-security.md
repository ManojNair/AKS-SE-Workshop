# Exercise 4: Networking and Security

## Overview

In this exercise, you will learn about networking and security concepts in AKS. You'll explore network policies, Azure CNI vs Kubenet, RBAC, and implement security best practices.

## Learning Objectives

By the end of this exercise, you will be able to:
- Understand Azure CNI vs Kubenet networking
- Configure and implement network policies
- Set up RBAC for access control
- Implement security best practices
- Configure network security groups
- Use Azure Key Vault integration
- Implement pod security policies

## Prerequisites

- Completed Exercise 3: Scaling and Resource Management
- AKS cluster with Azure CNI networking
- Understanding of basic networking concepts

## Exercise Steps

### Step 1: Prepare Your Environment

First, ensure you're connected to your AKS cluster and create a namespace for this exercise:

```bash
# Verify cluster connection
kubectl cluster-info

# Create namespace for this exercise
kubectl create namespace networking-workshop

# Set the namespace as default
kubectl config set-context --current --namespace=networking-workshop

# Check current network configuration
kubectl get nodes -o wide
```

### Step 2: Understand Your Network Configuration

Examine your current AKS network setup:

```bash
# Check cluster network configuration
az aks show \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --query networkProfile

# Check node pool network configuration
az aks nodepool list \
  --resource-group "rg-aks-workshop-01" \
  --cluster-name "aks-workshop-01-cluster" \
  --query "[].{name:name, networkProfile:networkProfile}"

# Check virtual network details
VNET_NAME=$(az aks show \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --query "agentPoolProfiles[0].vnetSubnetId" -o tsv | cut -d'/' -f9)

az network vnet show \
  --resource-group "rg-aks-workshop-01" \
  --name "$VNET_NAME" \
  --query "{addressSpace:addressSpace, subnets:subnets[].{name:name, addressPrefix:addressPrefix}}"
```

### Step 3: Deploy Test Applications

Create multiple applications to test networking and security:

```yaml
# Create test-applications.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
  namespace: networking-workshop
  labels:
    app: frontend
    tier: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      tier: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:latest
        ports:
        - containerPort: 80
        env:
        - name: BACKEND_URL
          value: "http://backend-app:8080"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: networking-workshop
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: frontend
    tier: frontend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
  namespace: networking-workshop
  labels:
    app: backend
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      tier: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
    spec:
      containers:
      - name: backend
        image: nginx:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: "database-app:5432"
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: networking-workshop
spec:
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: backend
    tier: backend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-app
  namespace: networking-workshop
  labels:
    app: database
    tier: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
      tier: database
  template:
    metadata:
      labels:
        app: database
        tier: database
    spec:
      containers:
      - name: database
        image: postgres:13
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          value: "password123"
---
apiVersion: v1
kind: Service
metadata:
  name: database-service
  namespace: networking-workshop
spec:
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: database
    tier: database
```

Deploy the applications:

```bash
# Apply the applications
kubectl apply -f test-applications.yaml

# Check all resources
kubectl get all --namespace=networking-workshop

# Check pod IP addresses
kubectl get pods -o wide --namespace=networking-workshop
```

### Step 4: Test Network Connectivity

Test connectivity between different application tiers:

```bash
# Test frontend to backend connectivity
kubectl exec -it $(kubectl get pods --namespace=networking-workshop -l app=frontend -o jsonpath='{.items[0].metadata.name}') \
  --namespace=networking-workshop \
  -- curl -I http://backend-service:8080

# Test backend to database connectivity
kubectl exec -it $(kubectl get pods --namespace=networking-workshop -l app=backend -o jsonpath='{.items[0].metadata.name}') \
  --namespace=networking-workshop \
  -- nc -zv database-service 5432

# Test frontend to database connectivity (should work without network policies)
kubectl exec -it $(kubectl get pods --namespace=networking-workshop -l app=frontend -o jsonpath='{.items[0].metadata.name}') \
  --namespace=networking-workshop \
  -- nc -zv database-service 5432
```

### Step 5: Implement Network Policies

Create network policies to control traffic between application tiers:

```yaml
# Create network-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: networking-workshop
spec:
  podSelector:
    matchLabels:
      app: frontend
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: networking-workshop
spec:
  podSelector:
    matchLabels:
      app: backend
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: networking-workshop
spec:
  podSelector:
    matchLabels:
      app: database
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

Apply the network policies:

```bash
# Apply network policies
kubectl apply -f network-policies.yaml

# Check network policies
kubectl get networkpolicies --namespace=networking-workshop

# Describe network policies
kubectl describe networkpolicy frontend-policy --namespace=networking-workshop
```

### Step 6: Test Network Policies

Test the network policies to ensure they're working correctly:

```bash
# Test frontend to backend (should work)
kubectl exec -it $(kubectl get pods --namespace=networking-workshop -l app=frontend -o jsonpath='{.items[0].metadata.name}') \
  --namespace=networking-workshop \
  -- curl -I http://backend-service:8080

# Test frontend to database (should be blocked)
kubectl exec -it $(kubectl get pods --namespace=networking-workshop -l app=frontend -o jsonpath='{.items[0].metadata.name}') \
  --namespace=networking-workshop \
  -- nc -zv database-service 5432

# Test backend to database (should work)
kubectl exec -it $(kubectl get pods --namespace=networking-workshop -l app=backend -o jsonpath='{.items[0].metadata.name}') \
  --namespace=networking-workshop \
  -- nc -zv database-service 5432
```

### Step 7: Set Up RBAC

Create RBAC roles and bindings for access control:

```yaml
# Create rbac-config.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: networking-workshop
  name: app-developer
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: networking-workshop
  name: app-reader
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-developer-binding
  namespace: networking-workshop
subjects:
- kind: User
  name: developer@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: app-developer
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-reader-binding
  namespace: networking-workshop
subjects:
- kind: User
  name: reader@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: app-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply RBAC configuration:

```bash
# Apply RBAC configuration
kubectl apply -f rbac-config.yaml

# Check roles and bindings
kubectl get roles --namespace=networking-workshop
kubectl get rolebindings --namespace=networking-workshop

# Describe roles
kubectl describe role app-developer --namespace=networking-workshop
```

### Step 8: Configure Azure Key Vault Integration

Set up Azure Key Vault integration for secrets management:

```bash
# Create Key Vault
az keyvault create \
  --resource-group "rg-aks-workshop-01" \
  --name "kv-aks-workshop-04" \
  --location "Australia East" \
  --sku standard

# Store a secret in Key Vault
az keyvault secret set \
  --vault-name "kv-aks-workshop-04" \
  --name "database-password" \
  --value "supersecretpassword123"

# Enable Key Vault integration with AKS
az aks enable-addons \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --addons azure-keyvault-secrets-provider

# Check Key Vault integration
kubectl get pods -n kube-system | grep secrets-store
```

### Step 9: Use Key Vault Secrets in Applications

Create a SecretProviderClass to access Key Vault secrets:

```yaml
# Create secret-provider-class.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: database-secrets
  namespace: networking-workshop
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: ""
    keyvaultName: "kv-aks-workshop-04"
    objects: |
      array:
        - |
          objectName: database-password
          objectType: secret
          objectVersion: ""
    tenantId: "YOUR-TENANT-ID"
  secretObjects:
  - data:
    - key: password
      objectName: database-password
    secretName: database-secret
    type: Opaque
```

Update the database deployment to use Key Vault secrets:

```yaml
# Update database-deployment-with-secrets.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-app-secure
  namespace: networking-workshop
  labels:
    app: database-secure
    tier: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database-secure
      tier: database
  template:
    metadata:
      labels:
        app: database-secure
        tier: database
    spec:
      containers:
      - name: database
        image: postgres:13
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: password
        volumeMounts:
        - name: secrets-store-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
      volumes:
      - name: secrets-store-inline
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "database-secrets"
```

### Step 10: Configure Network Security Groups

Create and configure NSGs for additional network security:

```bash
# Get the subnet ID
SUBNET_ID=$(az aks show \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --query "agentPoolProfiles[0].vnetSubnetId" -o tsv)

# Get the subnet name
SUBNET_NAME=$(echo $SUBNET_ID | cut -d'/' -f11)

# Get the VNet name
VNET_NAME=$(echo $SUBNET_ID | cut -d'/' -f9)

# Create NSG
az network nsg create \
  --resource-group "rg-aks-workshop-01" \
  --name "nsg-aks-workshop" \
  --location "Australia East"

# Add security rules
az network nsg rule create \
  --resource-group "rg-aks-workshop-01" \
  --nsg-name "nsg-aks-workshop" \
  --name "allow-https" \
  --protocol tcp \
  --priority 100 \
  --destination-port-range 443 \
  --access allow

az network nsg rule create \
  --resource-group "rg-aks-workshop-01" \
  --nsg-name "nsg-aks-workshop" \
  --name "allow-http" \
  --protocol tcp \
  --priority 110 \
  --destination-port-range 80 \
  --access allow

# Associate NSG with subnet
az network vnet subnet update \
  --resource-group "rg-aks-workshop-01" \
  --vnet-name "$VNET_NAME" \
  --name "$SUBNET_NAME" \
  --network-security-group "nsg-aks-workshop"
```

### Step 11: Implement Pod Security Policies

Create pod security policies to enforce security standards:

```yaml
# Create pod-security-policy.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted-psp
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
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: psp-restricted
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs: ['use']
  resourceNames:
  - restricted-psp
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: psp-restricted-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: psp-restricted
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:authenticated
```

### Step 12: Test Security Configurations

Test your security configurations:

```bash
# Test network policies
kubectl exec -it $(kubectl get pods --namespace=networking-workshop -l app=frontend -o jsonpath='{.items[0].metadata.name}') \
  --namespace=networking-workshop \
  -- nc -zv database-service 5432

# Test RBAC (if you have different users configured)
kubectl auth can-i create pods --namespace=networking-workshop
kubectl auth can-i delete deployments --namespace=networking-workshop

# Check Key Vault integration
kubectl get secret database-secret --namespace=networking-workshop

# Test pod security policies
kubectl run test-pod --image=nginx:latest --namespace=networking-workshop
kubectl describe pod test-pod --namespace=networking-workshop
```

## Understanding Networking and Security Concepts

### Azure CNI vs Kubenet

1. **Azure CNI**
   - Pods get IP addresses from VNet subnet
   - Better integration with Azure services
   - More complex but more powerful
   - Supports network policies

2. **Kubenet**
   - Pods get IP addresses from separate CIDR
   - Simpler but limited functionality
   - No network policy support
   - NAT required for external access

### Network Policies

Network policies control traffic between pods:
- **Ingress rules**: Control incoming traffic
- **Egress rules**: Control outgoing traffic
- **Pod selectors**: Target specific pods
- **Namespace selectors**: Target entire namespaces

### RBAC Best Practices

1. **Principle of Least Privilege**
   - Grant minimum required permissions
   - Use namespaces for isolation
   - Regular permission reviews

2. **Role Types**
   - **ClusterRole**: Cluster-wide permissions
   - **Role**: Namespace-specific permissions
   - **ClusterRoleBinding**: Bind cluster roles
   - **RoleBinding**: Bind namespace roles

### Exercise Questions

Answer these questions to reinforce your learning:

1. **What is the difference between Azure CNI and Kubenet?**
   - Azure CNI assigns VNet IPs to pods
   - Kubenet uses separate CIDR for pods
   - Azure CNI supports network policies

2. **How do network policies improve security?**
   - Control traffic between pods
   - Implement micro-segmentation
   - Reduce attack surface

3. **What are the benefits of using Azure Key Vault with AKS?**
   - Centralized secrets management
   - Integration with Azure AD
   - Audit and compliance features

## Hands-on Challenges

### Challenge 1: Multi-Namespace Network Policies
Create network policies that work across multiple namespaces:

```yaml
# Create namespace-specific policies
apiVersion: v1
kind: Namespace
metadata:
  name: frontend-ns
---
apiVersion: v1
kind: Namespace
metadata:
  name: backend-ns
---
apiVersion: v1
kind: Namespace
metadata:
  name: database-ns
```

### Challenge 2: Advanced RBAC Configuration
Create more granular RBAC roles:

```yaml
# Create role for monitoring only
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: networking-workshop
  name: monitoring-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list", "watch"]
```

### Challenge 3: Security Scanning
Implement security scanning for your containers:

```bash
# Install Trivy for vulnerability scanning
kubectl run trivy-scanner \
  --image=aquasec/trivy \
  --namespace=networking-workshop \
  --rm -it \
  -- trivy image nginx:latest
```

## Troubleshooting

### Common Issues

1. **Network Policies Not Working**
   ```bash
   # Check if network policies are enabled
   kubectl get pods -n kube-system | grep azure-npm
   
   # Check network policy events
   kubectl get events --namespace=networking-workshop
   ```

2. **RBAC Permission Issues**
   ```bash
   # Check user permissions
   kubectl auth can-i create pods --namespace=networking-workshop
   
   # Check role bindings
   kubectl get rolebindings --namespace=networking-workshop
   ```

3. **Key Vault Integration Issues**
   ```bash
   # Check secrets store CSI driver
   kubectl get pods -n kube-system | grep secrets-store
   
   # Check Key Vault provider logs
   kubectl logs -n kube-system deployment/azure-keyvault-secrets-provider
   ```

## Cleanup

Clean up resources from this exercise:

```bash
# Delete all resources in the namespace
kubectl delete namespace networking-workshop

# Delete Key Vault
az keyvault delete \
  --resource-group "rg-aks-workshop-01" \
  --name "kv-aks-workshop-04"

# Remove NSG association
az network vnet subnet update \
  --resource-group "rg-aks-workshop-01" \
  --vnet-name "$VNET_NAME" \
  --name "$SUBNET_NAME" \
  --remove networkSecurityGroup

# Delete NSG
az network nsg delete \
  --resource-group "rg-aks-workshop-01" \
  --name "nsg-aks-workshop"
```

## Summary

In this exercise, you have:
- ✅ Understood Azure CNI vs Kubenet networking
- ✅ Configured and implemented network policies
- ✅ Set up RBAC for access control
- ✅ Implemented security best practices
- ✅ Configured network security groups
- ✅ Used Azure Key Vault integration
- ✅ Implemented pod security policies

## Next Steps

You're now ready to move to **Exercise 5: Monitoring and Logging** where you'll learn about Azure Monitor, Log Analytics, and custom metrics.

## Additional Resources

- [AKS Networking](https://docs.microsoft.com/en-us/azure/aks/concepts-network)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Azure Key Vault](https://docs.microsoft.com/en-us/azure/key-vault/)
- [Pod Security Policies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) 