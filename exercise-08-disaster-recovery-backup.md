# Exercise 8: Disaster Recovery and Backup for AKS

## Overview

This exercise covers disaster recovery and backup strategies for Azure Kubernetes Service (AKS) clusters. You'll learn how to implement backup solutions, configure disaster recovery, and ensure business continuity for your containerized applications.

## Learning Objectives

- Understand AKS disaster recovery concepts
- Implement backup strategies for applications and data
- Configure cross-region disaster recovery
- Set up automated backup and restore procedures
- Test disaster recovery scenarios

## Prerequisites

- Completed Exercises 1-7
- Azure subscription with appropriate permissions
- Azure CLI installed and configured
- kubectl installed
- Helm installed (optional)

## Duration

- **Estimated Time**: 2-3 hours
- **Difficulty**: Advanced

---

## Part 1: Understanding AKS Disaster Recovery

### 1.1 AKS Disaster Recovery Concepts

AKS disaster recovery involves protecting your applications and data from various failure scenarios:

**Key Concepts:**
- **Application Backup**: Backing up application data and configurations
- **Infrastructure Recovery**: Recreating AKS clusters in different regions
- **Data Replication**: Ensuring data availability across regions
- **RTO (Recovery Time Objective)**: Maximum acceptable downtime
- **RPO (Recovery Point Objective)**: Maximum acceptable data loss

**Common Failure Scenarios:**
- Regional Azure outages
- AKS cluster failures
- Application data corruption
- Network connectivity issues
- Human error

### 1.2 AKS Built-in Resilience Features

AKS provides several built-in resilience features:

```bash
# Check AKS cluster availability zones
az aks show --resource-group myResourceGroup --name myAKSCluster --query "availabilityZones"

# Check node pool configuration
az aks nodepool list --resource-group myResourceGroup --cluster-name myAKSCluster --output table
```

**Windows PowerShell:**
```powershell
# Check AKS cluster availability zones
az aks show --resource-group myResourceGroup --name myAKSCluster --query "availabilityZones"

# Check node pool configuration
az aks nodepool list --resource-group myResourceGroup --cluster-name myAKSCluster --output table
```

---

## Part 2: Application Data Backup Strategies

### 2.1 Azure Storage Account Backup

Create a storage account for backup data:

```bash
# Create storage account for backups
az storage account create \
  --resource-group myResourceGroup \
  --name myaksbackupstorage \
  --location australiaeast \
  --sku Standard_LRS \
  --encryption-services blob

# Get storage account key
STORAGE_KEY=$(az storage account keys list \
  --resource-group myResourceGroup \
  --account-name myaksbackupstorage \
  --query '[0].value' -o tsv)

# Create container for backups
az storage container create \
  --account-name myaksbackupstorage \
  --account-key $STORAGE_KEY \
  --name aks-backups
```

**Windows PowerShell:**
```powershell
# Create storage account for backups
az storage account create `
  --resource-group myResourceGroup `
  --name myaksbackupstorage `
  --location australiaeast `
  --sku Standard_LRS `
  --encryption-services blob

# Get storage account key
$STORAGE_KEY = az storage account keys list `
  --resource-group myResourceGroup `
  --account-name myaksbackupstorage `
  --query '[0].value' -o tsv

# Create container for backups
az storage container create `
  --account-name myaksbackupstorage `
  --account-key $STORAGE_KEY `
  --name aks-backups
```

### 2.2 Velero Backup Solution

Install Velero for Kubernetes backup:

```bash
# Download Velero CLI
wget https://github.com/vmware-tanzu/velero/releases/download/v1.11.0/velero-v1.11.0-linux-amd64.tar.gz
tar -xzf velero-v1.11.0-linux-amd64.tar.gz
sudo mv velero /usr/local/bin/

# Create Velero service principal
az ad sp create-for-rbac --name velero-backup --role contributor --scopes /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/myResourceGroup

# Install Velero in AKS cluster
velero install \
  --provider azure \
  --plugins velero/velero-plugin-for-microsoft-azure:v1.6.0 \
  --bucket aks-backups \
  --secret-file ./credentials-velero \
  --backup-location-config resourceGroup=myResourceGroup,storageAccount=myaksbackupstorage \
  --snapshot-location-config apiTimeout=5m \
  --use-volume-snapshots=false
```

**Windows PowerShell:**
```powershell
# Download Velero CLI (using Chocolatey or download manually)
# choco install velero

# Or download manually from: https://github.com/vmware-tanzu/velero/releases

# Create Velero service principal
az ad sp create-for-rbac --name velero-backup --role contributor --scopes /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/myResourceGroup

# Install Velero in AKS cluster
velero install `
  --provider azure `
  --plugins velero/velero-plugin-for-microsoft-azure:v1.6.0 `
  --bucket aks-backups `
  --secret-file ./credentials-velero `
  --backup-location-config resourceGroup=myResourceGroup,storageAccount=myaksbackupstorage `
  --snapshot-location-config apiTimeout=5m `
  --use-volume-snapshots=false
```

### 2.3 Create Backup Schedule

```bash
# Create scheduled backup for all namespaces
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 168h \
  --include-namespaces default,kube-system

# Create backup schedule for specific applications
velero schedule create app-backup \
  --schedule="0 3 * * *" \
  --ttl 72h \
  --include-namespaces myapp \
  --include-resources deployments,services,configmaps,secrets,pvc
```

**Windows PowerShell:**
```powershell
# Create scheduled backup for all namespaces
velero schedule create daily-backup `
  --schedule="0 2 * * *" `
  --ttl 168h `
  --include-namespaces default,kube-system

# Create backup schedule for specific applications
velero schedule create app-backup `
  --schedule="0 3 * * *" `
  --ttl 72h `
  --include-namespaces myapp `
  --include-resources deployments,services,configmaps,secrets,pvc
```

---

## Part 3: Cross-Region Disaster Recovery

### 3.1 Secondary Region Setup

Create a secondary AKS cluster in a different region:

```bash
# Create resource group in secondary region
az group create \
  --name myResourceGroup-DR \
  --location australiasoutheast

# Create AKS cluster in secondary region
az aks create \
  --resource-group myResourceGroup-DR \
  --name myAKSCluster-DR \
  --node-count 2 \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --location australiasoutheast \
  --node-vm-size Standard_DS2_v2

# Get credentials for secondary cluster
az aks get-credentials \
  --resource-group myResourceGroup-DR \
  --name myAKSCluster-DR \
  --overwrite-existing
```

**Windows PowerShell:**
```powershell
# Create resource group in secondary region
az group create `
  --name myResourceGroup-DR `
  --location australiasoutheast

# Create AKS cluster in secondary region
az aks create `
  --resource-group myResourceGroup-DR `
  --name myAKSCluster-DR `
  --node-count 2 `
  --enable-addons monitoring `
  --generate-ssh-keys `
  --location australiasoutheast `
  --node-vm-size Standard_DS2_v2

# Get credentials for secondary cluster
az aks get-credentials `
  --resource-group myResourceGroup-DR `
  --name myAKSCluster-DR `
  --overwrite-existing
```

### 3.2 Configure Cross-Region Backup

```bash
# Create storage account in secondary region
az storage account create \
  --resource-group myResourceGroup-DR \
  --name myaksbackupstorage-dr \
  --location australiasoutheast \
  --sku Standard_LRS \
  --encryption-services blob

# Configure Velero for cross-region backup
velero backup-location create secondary-location \
  --provider azure \
  --bucket aks-backups \
  --config resourceGroup=myResourceGroup-DR,storageAccount=myaksbackupstorage-dr
```

**Windows PowerShell:**
```powershell
# Create storage account in secondary region
az storage account create `
  --resource-group myResourceGroup-DR `
  --name myaksbackupstorage-dr `
  --location australiasoutheast `
  --sku Standard_LRS `
  --encryption-services blob

# Configure Velero for cross-region backup
velero backup-location create secondary-location `
  --provider azure `
  --bucket aks-backups `
  --config resourceGroup=myResourceGroup-DR,storageAccount=myaksbackupstorage-dr
```

---

## Part 4: Database Backup and Recovery

### 4.1 Azure Database for PostgreSQL Backup

If using Azure Database for PostgreSQL:

```bash
# Create PostgreSQL server with backup enabled
az postgres flexible-server create \
  --resource-group myResourceGroup \
  --name mypostgresqlserver \
  --location australiaeast \
  --admin-user postgresadmin \
  --admin-password "YourPassword123!" \
  --sku-name Standard_B1ms \
  --version 13 \
  --storage-size 32 \
  --backup-retention 7 \
  --geo-redundant-backup Enabled

# Configure automated backups
az postgres flexible-server parameter set \
  --resource-group myResourceGroup \
  --server-name mypostgresqlserver \
  --name log_checkpoints \
  --value on
```

**Windows PowerShell:**
```powershell
# Create PostgreSQL server with backup enabled
az postgres flexible-server create `
  --resource-group myResourceGroup `
  --name mypostgresqlserver `
  --location australiaeast `
  --admin-user postgresadmin `
  --admin-password "YourPassword123!" `
  --sku-name Standard_B1ms `
  --version 13 `
  --storage-size 32 `
  --backup-retention 7 `
  --geo-redundant-backup Enabled

# Configure automated backups
az postgres flexible-server parameter set `
  --resource-group myResourceGroup `
  --server-name mypostgresqlserver `
  --name log_checkpoints `
  --value on
```

### 4.2 Azure Cosmos DB Backup

For Cosmos DB applications:

```bash
# Create Cosmos DB account with backup policy
az cosmosdb create \
  --resource-group myResourceGroup \
  --name mycosmosdbaccount \
  --locations regionName=australiaeast failoverPriority=0 isZoneRedundant=false \
  --capabilities EnableServerless \
  --backup-policy-type Continuous \
  --enable-multiple-write-locations false
```

**Windows PowerShell:**
```powershell
# Create Cosmos DB account with backup policy
az cosmosdb create `
  --resource-group myResourceGroup `
  --name mycosmosdbaccount `
  --locations regionName=australiaeast failoverPriority=0 isZoneRedundant=false `
  --capabilities EnableServerless `
  --backup-policy-type Continuous `
  --enable-multiple-write-locations false
```

---

## Part 5: Testing Disaster Recovery

### 5.1 Manual Backup Test

```bash
# Create manual backup
velero backup create manual-backup-test \
  --include-namespaces default \
  --include-resources deployments,services,configmaps,secrets

# Check backup status
velero backup describe manual-backup-test

# List all backups
velero backup get
```

**Windows PowerShell:**
```powershell
# Create manual backup
velero backup create manual-backup-test `
  --include-namespaces default `
  --include-resources deployments,services,configmaps,secrets

# Check backup status
velero backup describe manual-backup-test

# List all backups
velero backup get
```

### 5.2 Restore Test

```bash
# Test restore to a new namespace
velero restore create --from-backup manual-backup-test \
  --namespace-mappings default:default-restored

# Check restore status
velero restore describe manual-backup-test-20231201120000

# Verify restored resources
kubectl get all -n default-restored
```

**Windows PowerShell:**
```powershell
# Test restore to a new namespace
velero restore create --from-backup manual-backup-test `
  --namespace-mappings default:default-restored

# Check restore status
velero restore describe manual-backup-test-20231201120000

# Verify restored resources
kubectl get all -n default-restored
```

### 5.3 Cross-Region Recovery Test

```bash
# Switch to secondary cluster context
kubectl config use-context myAKSCluster-DR

# Restore backup to secondary region
velero restore create --from-backup manual-backup-test \
  --namespace-mappings default:default-dr

# Verify application functionality
kubectl get pods -n default-dr
kubectl logs -n default-dr deployment/myapp
```

**Windows PowerShell:**
```powershell
# Switch to secondary cluster context
kubectl config use-context myAKSCluster-DR

# Restore backup to secondary region
velero restore create --from-backup manual-backup-test `
  --namespace-mappings default:default-dr

# Verify application functionality
kubectl get pods -n default-dr
kubectl logs -n default-dr deployment/myapp
```

---

## Part 6: Monitoring and Alerting

### 6.1 Backup Monitoring

```bash
# Create backup monitoring dashboard
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: backup-monitoring
  namespace: velero
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "Velero Backup Monitoring",
        "panels": [
          {
            "title": "Backup Success Rate",
            "type": "stat",
            "targets": [
              {
                "expr": "velero_backup_total{result=\"success\"} / velero_backup_total"
              }
            ]
          }
        ]
      }
    }
EOF
```

**Windows PowerShell:**
```powershell
# Create backup monitoring dashboard
@"
apiVersion: v1
kind: ConfigMap
metadata:
  name: backup-monitoring
  namespace: velero
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "Velero Backup Monitoring",
        "panels": [
          {
            "title": "Backup Success Rate",
            "type": "stat",
            "targets": [
              {
                "expr": "velero_backup_total{result=\"success\"} / velero_backup_total"
              }
            ]
          }
        ]
      }
    }
"@ | kubectl apply -f -
```

### 6.2 Backup Alerting

```bash
# Create backup failure alert
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1alpha1
kind: PrometheusRule
metadata:
  name: velero-backup-alerts
  namespace: velero
spec:
  groups:
  - name: velero-backup
    rules:
    - alert: VeleroBackupFailed
      expr: velero_backup_total{result="failure"} > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Velero backup failed"
        description: "Backup {{ $labels.backup_name }} has failed"
EOF
```

**Windows PowerShell:**
```powershell
# Create backup failure alert
@"
apiVersion: monitoring.coreos.com/v1alpha1
kind: PrometheusRule
metadata:
  name: velero-backup-alerts
  namespace: velero
spec:
  groups:
  - name: velero-backup
    rules:
  - alert: VeleroBackupFailed
    expr: velero_backup_total{result="failure"} > 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Velero backup failed"
      description: "Backup {{ `$labels.backup_name }} has failed"
"@ | kubectl apply -f -
```

---

## Part 7: Disaster Recovery Runbook

### 7.1 Create Recovery Procedures

Create a disaster recovery runbook:

```bash
# Create recovery runbook
cat <<EOF > disaster-recovery-runbook.md
# AKS Disaster Recovery Runbook

## Primary Region Failure

### Step 1: Assess the Situation
- Check Azure status page
- Verify cluster health
- Determine failure scope

### Step 2: Activate Secondary Region
- Switch traffic to secondary region
- Update DNS records
- Notify stakeholders

### Step 3: Restore Applications
- Restore from latest backup
- Verify application functionality
- Update configuration if needed

### Step 4: Monitor and Validate
- Monitor application performance
- Validate data integrity
- Document incident details

## Recovery Time Objectives (RTO)
- Critical applications: 15 minutes
- Standard applications: 1 hour
- Non-critical applications: 4 hours

## Recovery Point Objectives (RPO)
- Database: 5 minutes
- Application state: 15 minutes
- Configuration: 1 hour
EOF
```

**Windows PowerShell:**
```powershell
# Create recovery runbook
@"
# AKS Disaster Recovery Runbook

## Primary Region Failure

### Step 1: Assess the Situation
- Check Azure status page
- Verify cluster health
- Determine failure scope

### Step 2: Activate Secondary Region
- Switch traffic to secondary region
- Update DNS records
- Notify stakeholders

### Step 3: Restore Applications
- Restore from latest backup
- Verify application functionality
- Update configuration if needed

### Step 4: Monitor and Validate
- Monitor application performance
- Validate data integrity
- Document incident details

## Recovery Time Objectives (RTO)
- Critical applications: 15 minutes
- Standard applications: 1 hour
- Non-critical applications: 4 hours

## Recovery Point Objectives (RPO)
- Database: 5 minutes
- Application state: 15 minutes
- Configuration: 1 hour
"@ | Out-File -FilePath disaster-recovery-runbook.md -Encoding UTF8
```

---

## Challenges and Exercises

### Challenge 1: Multi-Region Application Deployment
Deploy your application to both primary and secondary regions and configure traffic routing between them.

### Challenge 2: Automated Failover Testing
Create an automated test that simulates a regional failure and validates the failover process.

### Challenge 3: Backup Policy Optimization
Design backup policies that balance cost, performance, and recovery objectives for different application tiers.

### Challenge 4: Data Consistency Validation
Implement automated tests to validate data consistency after backup and restore operations.

---

## Troubleshooting

### Common Issues and Solutions

**Backup Failures:**
```bash
# Check Velero logs
kubectl logs -n velero deployment/velero

# Verify storage account permissions
az storage account show --name myaksbackupstorage --query "identity"
```

**Restore Failures:**
```bash
# Check restore logs
velero restore logs RESTORE_NAME

# Verify namespace mappings
velero restore describe RESTORE_NAME
```

**Cross-Region Issues:**
```bash
# Check network connectivity
kubectl run test-connectivity --image=busybox --rm -it --restart=Never -- nslookup mycosmosdbaccount.documents.azure.com

# Verify service principal permissions
az role assignment list --assignee velero-backup --scope /subscriptions/YOUR_SUBSCRIPTION_ID
```

---

## Cleanup

### Remove Backup Resources

```bash
# Delete Velero
kubectl delete namespace velero

# Delete backup storage accounts
az storage account delete \
  --resource-group myResourceGroup \
  --name myaksbackupstorage \
  --yes

az storage account delete \
  --resource-group myResourceGroup-DR \
  --name myaksbackupstorage-dr \
  --yes

# Delete secondary cluster
az aks delete \
  --resource-group myResourceGroup-DR \
  --name myAKSCluster-DR \
  --yes

# Delete secondary resource group
az group delete \
  --name myResourceGroup-DR \
  --yes
```

**Windows PowerShell:**
```powershell
# Delete Velero
kubectl delete namespace velero

# Delete backup storage accounts
az storage account delete `
  --resource-group myResourceGroup `
  --name myaksbackupstorage `
  --yes

az storage account delete `
  --resource-group myResourceGroup-DR `
  --name myaksbackupstorage-dr `
  --yes

# Delete secondary cluster
az aks delete `
  --resource-group myResourceGroup-DR `
  --name myAKSCluster-DR `
  --yes

# Delete secondary resource group
az group delete `
  --name myResourceGroup-DR `
  --yes
```

---

## Summary

In this exercise, you learned:

✅ **Disaster Recovery Concepts**: Understanding RTO, RPO, and failure scenarios  
✅ **Backup Strategies**: Implementing Velero for Kubernetes backup and restore  
✅ **Cross-Region Recovery**: Setting up secondary regions and backup replication  
✅ **Database Backup**: Configuring backup for Azure databases  
✅ **Testing Procedures**: Creating and testing backup and restore procedures  
✅ **Monitoring**: Setting up alerts and monitoring for backup operations  
✅ **Recovery Runbooks**: Creating documented recovery procedures  

## Next Steps

- Implement backup policies for production workloads
- Set up automated disaster recovery testing
- Configure monitoring and alerting for backup operations
- Document and train team members on recovery procedures
- Regularly test and update disaster recovery plans

## Additional Resources

- [Velero Documentation](https://velero.io/docs/)
- [Azure Backup for AKS](https://docs.microsoft.com/en-us/azure/backup/azure-kubernetes-service-cluster-backup)
- [AKS Disaster Recovery Best Practices](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-multi-region)
- [Azure Site Recovery](https://docs.microsoft.com/en-us/azure/site-recovery/)
- [Kubernetes Backup Strategies](https://kubernetes.io/docs/concepts/cluster-administration/backup/)

---

**Congratulations!** You've completed the AKS Workshop. You now have comprehensive knowledge of Azure Kubernetes Service, from basic cluster setup to advanced disaster recovery strategies. 