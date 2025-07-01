# Exercise 6: CI/CD with AKS

## Overview

In this exercise, you will learn how to set up continuous integration and continuous deployment (CI/CD) for your AKS cluster using GitHub Actions.

## Learning Objectives

- Set up GitHub Actions for AKS deployments
- Create automated CI/CD pipelines
- Integrate with Azure Container Registry
- Implement deployment strategies
- Set up automated testing

## Prerequisites

- Completed Exercise 5: Monitoring and Logging
- GitHub account with repository access
- Azure Container Registry (ACR)

## Exercise Steps

### Step 1: Prepare Your Environment

```bash
# Create Azure Container Registry
az acr create \
  --resource-group "rg-aks-workshop-01" \
  --name "acrworkshop06" \
  --sku Basic \
  --admin-enabled true

# Get ACR login server
ACR_LOGIN_SERVER=$(az acr show \
  --resource-group "rg-aks-workshop-01" \
  --name "acrworkshop06" \
  --query loginServer -o tsv)

echo "ACR Login Server: $ACR_LOGIN_SERVER"

# Attach ACR to AKS
az aks update \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --attach-acr "acrworkshop06"

# Create namespace for CI/CD
kubectl create namespace cicd-workshop
kubectl config set-context --current --namespace=cicd-workshop
```

### Step 2: Create Sample Application

```bash
# Create application directory
mkdir -p sample-app
cd sample-app

# Create Dockerfile
cat <<EOF > Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF

# Create index.html
cat <<EOF > index.html
<!DOCTYPE html>
<html>
<head>
    <title>AKS Workshop App</title>
</head>
<body>
    <h1>Welcome to AKS Workshop</h1>
    <p>This is a sample application for CI/CD testing.</p>
    <p>Version: 1.0.0</p>
</body>
</html>
EOF

# Create Kubernetes manifests
mkdir -p k8s

cat <<EOF > k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: cicd-workshop
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: acrworkshop06.azurecr.io/sample-app:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF

cat <<EOF > k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app-service
  namespace: cicd-workshop
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: sample-app
EOF
```

### Step 3: Create GitHub Actions Workflow

```yaml
# Create .github/workflows/ci-cd.yaml
name: CI/CD Pipeline with OIDC

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  ACR_NAME: acrworkshop06
  AKS_CLUSTER_NAME: aks-workshop-01-cluster
  RESOURCE_GROUP: rg-aks-workshop-01
  NAMESPACE: cicd-workshop

permissions:
  id-token: write  # Required for OIDC token exchange
  contents: read   # Required for actions/checkout

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Azure Login for ACR
      uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: Log in to Azure Container Registry
      run: |
        az acr login --name ${{ env.ACR_NAME }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ${{ env.ACR_NAME }}.azurecr.io/sample-app:${{ github.sha }}
          ${{ env.ACR_NAME }}.azurecr.io/sample-app:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      id-token: write  # Required for OIDC token exchange
      contents: read   # Required for actions/checkout
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Azure Login
      uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: Get AKS credentials
      run: |
        az aks get-credentials \
          --resource-group ${{ env.RESOURCE_GROUP }} \
          --name ${{ env.AKS_CLUSTER_NAME }} \
          --overwrite-existing

    - name: Deploy to AKS
      run: |
        # Update image tag in deployment
        sed -i "s|:latest|:${{ github.sha }}|g" k8s/deployment.yaml
        
        # Apply Kubernetes manifests
        kubectl apply -f k8s/ -n ${{ env.NAMESPACE }}
        
        # Wait for deployment to be ready
        kubectl rollout status deployment/sample-app -n ${{ env.NAMESPACE }}

    - name: Run tests
      run: |
        # Wait for service to be ready
        kubectl wait --for=condition=ready pod -l app=sample-app -n ${{ env.NAMESPACE }} --timeout=300s
        
        # Get service IP
        SERVICE_IP=$(kubectl get service sample-app-service -n ${{ env.NAMESPACE }} -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
        
        # Wait for IP to be assigned
        while [ -z "$SERVICE_IP" ]; do
          sleep 10
          SERVICE_IP=$(kubectl get service sample-app-service -n ${{ env.NAMESPACE }} -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
        done
        
        # Test application
        curl -f http://$SERVICE_IP || exit 1

    - name: Verify deployment
      run: |
        echo "Deployment verification:"
        kubectl get pods -n ${{ env.NAMESPACE }}
        kubectl get services -n ${{ env.NAMESPACE }}
        kubectl get deployments -n ${{ env.NAMESPACE }}
```

### Step 4: Set Up OpenID Connect (OIDC) with Azure

Instead of storing long-lived secrets, we'll use OpenID Connect (OIDC) for secure authentication with Azure. This approach is more secure and follows security best practices.

#### 4.1 Create Azure AD Application and Service Principal

```bash
# Create Azure AD application
az ad app create --display-name "aks-workshop-oidc"

# Get the application ID
APP_ID=$(az ad app list --display-name "aks-workshop-oidc" --query "[0].appId" -o tsv)
echo "Application ID: $APP_ID"

# Create service principal
az ad sp create --id $APP_ID

# Get the service principal ID
SP_ID=$(az ad sp list --display-name "aks-workshop-oidc" --query "[0].id" -o tsv)
echo "Service Principal ID: $SP_ID"

# Assign Contributor role to the service principal
az role assignment create \
  --assignee $SP_ID \
  --role Contributor \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-aks-workshop-01"
```

**Windows PowerShell:**
```powershell
# Create Azure AD application
az ad app create --display-name "aks-workshop-oidc"

# Get the application ID
$APP_ID = az ad app list --display-name "aks-workshop-oidc" --query "[0].appId" -o tsv
Write-Host "Application ID: $APP_ID"

# Create service principal
az ad sp create --id $APP_ID

# Get the service principal ID
$SP_ID = az ad sp list --display-name "aks-workshop-oidc" --query "[0].id" -o tsv
Write-Host "Service Principal ID: $SP_ID"

# Assign Contributor role to the service principal
az role assignment create `
  --assignee $SP_ID `
  --role Contributor `
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-aks-workshop-01"
```

#### 4.2 Configure Federated Credentials

```bash
# Get your GitHub repository details
GITHUB_REPO="yourusername/aks-workshop"  # Replace with your repository
GITHUB_BRANCH="main"  # Replace with your main branch

# Create federated credential for GitHub Actions
az ad app federated-credential create \
  --id $APP_ID \
  --parameters "{\"name\":\"github-actions\",\"issuer\":\"https://token.actions.githubusercontent.com\",\"subject\":\"repo:$GITHUB_REPO:ref:refs/heads/$GITHUB_BRANCH\",\"audience\":\"api://AzureADTokenExchange\",\"description\":\"GitHub Actions OIDC\"}"

# Create federated credential for pull requests (optional)
az ad app federated-credential create \
  --id $APP_ID \
  --parameters "{\"name\":\"github-actions-pr\",\"issuer\":\"https://token.actions.githubusercontent.com\",\"subject\":\"repo:$GITHUB_REPO:pull_request\",\"audience\":\"api://AzureADTokenExchange\",\"description\":\"GitHub Actions OIDC for PRs\"}"
```

**Windows PowerShell:**
```powershell
# Get your GitHub repository details
$GITHUB_REPO = "yourusername/aks-workshop"  # Replace with your repository
$GITHUB_BRANCH = "main"  # Replace with your main branch

# Create federated credential for GitHub Actions
az ad app federated-credential create `
  --id $APP_ID `
  --parameters "{\"name\":\"github-actions\",\"issuer\":\"https://token.actions.githubusercontent.com\",\"subject\":\"repo:$GITHUB_REPO:ref:refs/heads/$GITHUB_BRANCH\",\"audience\":\"api://AzureADTokenExchange\",\"description\":\"GitHub Actions OIDC\"}"

# Create federated credential for pull requests (optional)
az ad app federated-credential create `
  --id $APP_ID `
  --parameters "{\"name\":\"github-actions-pr\",\"issuer\":\"https://token.actions.githubusercontent.com\",\"subject\":\"repo:$GITHUB_REPO:pull_request\",\"audience\":\"api://AzureADTokenExchange\",\"description\":\"GitHub Actions OIDC for PRs\"}"
```

#### 4.3 Set Up GitHub Secrets

You only need to store the Azure configuration details, not sensitive credentials:

```bash
# Get Azure subscription and tenant details
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)

echo "Subscription ID: $SUBSCRIPTION_ID"
echo "Tenant ID: $TENANT_ID"
echo "Client ID (Application ID): $APP_ID"

# Store these values as GitHub secrets:
# AZURE_CLIENT_ID: $APP_ID
# AZURE_TENANT_ID: $TENANT_ID
# AZURE_SUBSCRIPTION_ID: $SUBSCRIPTION_ID
```

**Windows PowerShell:**
```powershell
# Get Azure subscription and tenant details
$SUBSCRIPTION_ID = az account show --query id -o tsv
$TENANT_ID = az account show --query tenantId -o tsv

Write-Host "Subscription ID: $SUBSCRIPTION_ID"
Write-Host "Tenant ID: $TENANT_ID"
Write-Host "Client ID (Application ID): $APP_ID"

# Store these values as GitHub secrets:
# AZURE_CLIENT_ID: $APP_ID
# AZURE_TENANT_ID: $TENANT_ID
# AZURE_SUBSCRIPTION_ID: $SUBSCRIPTION_ID
```

**GitHub Secrets to Configure:**
1. Go to your GitHub repository → Settings → Secrets and variables → Actions
2. Add the following repository secrets:
   - `AZURE_CLIENT_ID`: The Application ID from step 4.1
   - `AZURE_TENANT_ID`: Your Azure tenant ID
   - `AZURE_SUBSCRIPTION_ID`: Your Azure subscription ID

### Step 5: Test the Pipeline

```bash
# Initialize git repository
git init
git add .
git commit -m "Initial commit: Sample application for AKS CI/CD"

# Create .gitignore
cat <<EOF > .gitignore
node_modules/
*.log
.env
.DS_Store
EOF

# Create GitHub Actions workflow directory
mkdir -p .github/workflows

# Push to GitHub (replace with your repository URL)
git remote add origin https://github.com/yourusername/aks-workshop.git
git push -u origin main
```

## Understanding CI/CD Concepts

### GitHub Actions Components

1. **Workflows**
   - YAML files defining CI/CD processes
   - Triggered by events (push, PR, manual)

2. **Jobs**
   - Run on specific runners
   - Can run in parallel or sequentially

3. **Steps**
   - Individual tasks within jobs
   - Can use actions or run commands

### Deployment Strategies

1. **Rolling Update**
   - Update pods one by one
   - Maintain service availability

2. **Blue-Green Deployment**
   - Two identical environments
   - Switch traffic between them

3. **Canary Deployment**
   - Gradual rollout to users
   - Monitor performance and errors

### OpenID Connect (OIDC) Security Benefits

Using OIDC instead of long-lived secrets provides several security advantages:

1. **No Long-lived Secrets**: Eliminates the need to store sensitive credentials in GitHub secrets
2. **Short-lived Tokens**: Access tokens are automatically generated and expire quickly
3. **Fine-grained Control**: Federated credentials can be scoped to specific repositories, branches, or workflows
4. **Audit Trail**: All token exchanges are logged in Azure AD audit logs
5. **Automatic Rotation**: No manual secret rotation required

**How OIDC Works:**
1. GitHub Actions requests an ID token from GitHub's OIDC provider
2. The ID token is exchanged with Azure AD for an access token
3. The access token is used to authenticate with Azure services
4. Tokens expire automatically, reducing the attack surface

For more information, see the [GitHub documentation on OIDC with Azure](https://docs.github.com/en/actions/how-tos/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-azure).

## Hands-on Challenges

### Challenge 1: Multi-Environment Pipeline
Create a pipeline that deploys to multiple environments (dev, staging, prod).

### Challenge 2: Automated Testing
Add comprehensive testing including unit tests and integration tests.

### Challenge 3: Security Integration
Integrate security scanning and vulnerability assessment.

## Troubleshooting

### Common Issues

1. **Authentication Failures**
   ```bash
   # Verify service principal permissions
   az role assignment list --assignee <service-principal-id>
   ```

2. **Image Build Failures**
   ```bash
   # Check Dockerfile syntax
   docker build --no-cache .
   ```

3. **Deployment Failures**
   ```bash
   # Check pod events
   kubectl describe pod <pod-name> -n cicd-workshop
   ```

## Cleanup

```bash
# Delete all resources in the namespace
kubectl delete namespace cicd-workshop

# Delete ACR
az acr delete \
  --resource-group "rg-aks-workshop-01" \
  --name "acrworkshop06"

# Clean up OIDC resources
# Get the application ID
APP_ID=$(az ad app list --display-name "aks-workshop-oidc" --query "[0].appId" -o tsv)

# Delete federated credentials
az ad app federated-credential delete \
  --id $APP_ID \
  --federated-credential-id "github-actions"

az ad app federated-credential delete \
  --id $APP_ID \
  --federated-credential-id "github-actions-pr"

# Delete the Azure AD application
az ad app delete --id $APP_ID

# Clean up local files
rm -rf sample-app
```

**Windows PowerShell:**
```powershell
# Delete all resources in the namespace
kubectl delete namespace cicd-workshop

# Delete ACR
az acr delete `
  --resource-group "rg-aks-workshop-01" `
  --name "acrworkshop06"

# Clean up OIDC resources
# Get the application ID
$APP_ID = az ad app list --display-name "aks-workshop-oidc" --query "[0].appId" -o tsv

# Delete federated credentials
az ad app federated-credential delete `
  --id $APP_ID `
  --federated-credential-id "github-actions"

az ad app federated-credential delete `
  --id $APP_ID `
  --federated-credential-id "github-actions-pr"

# Delete the Azure AD application
az ad app delete --id $APP_ID

# Clean up local files
Remove-Item -Recurse -Force sample-app
```

## Summary

In this exercise, you have:
- ✅ Set up GitHub Actions for AKS deployments with OpenID Connect (OIDC)
- ✅ Created automated CI/CD pipelines using secure authentication
- ✅ Integrated with Azure Container Registry using OIDC
- ✅ Implemented deployment strategies with enhanced security
- ✅ Set up automated testing with short-lived tokens
- ✅ Configured federated credentials for secure Azure authentication

## Next Steps

You're now ready to move to **Exercise 7: Advanced AKS Features** where you'll learn about virtual nodes, GPU workloads, and Windows containers.

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-registry/)
- [AKS Deployment Strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [GitHub Actions for Azure](https://github.com/azure/actions) 