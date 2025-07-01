# Exercise 5: Monitoring and Logging

## Overview

In this exercise, you will learn how to set up comprehensive monitoring and logging for your AKS cluster using Azure Monitor, Log Analytics, and custom metrics.

## Learning Objectives

- Set up Azure Monitor for containers
- Configure Log Analytics workspace
- Create custom metrics and dashboards
- Set up alerting and notifications
- Monitor cluster and application performance
- Analyze logs and troubleshoot issues

## Prerequisites

- Completed Exercise 4: Networking and Security
- AKS cluster with monitoring add-on enabled

## Exercise Steps

### Step 1: Prepare Your Environment

```bash
# Verify cluster connection
kubectl cluster-info

# Create namespace for this exercise
kubectl create namespace monitoring-workshop

# Set the namespace as default
kubectl config set-context --current --namespace=monitoring-workshop

# Check if monitoring add-on is enabled
az aks show \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --query "addonProfiles.omsagent.enabled"
```

### Step 2: Deploy Test Applications

Create applications to generate monitoring data:

```yaml
# Create monitoring-test-apps.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: high-cpu-app
  namespace: monitoring-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: high-cpu-app
  template:
    metadata:
      labels:
        app: high-cpu-app
    spec:
      containers:
      - name: high-cpu-app
        image: busybox:latest
        command: ["/bin/sh"]
        args: ["-c", "while true; do dd if=/dev/zero of=/dev/null bs=1M count=1000; sleep 1; done"]
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "500m"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: high-memory-app
  namespace: monitoring-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: high-memory-app
  template:
    metadata:
      labels:
        app: high-memory-app
    spec:
      containers:
      - name: high-memory-app
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: high-memory-service
  namespace: monitoring-workshop
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: high-memory-app
```

Deploy the applications:

```bash
# Apply the applications
kubectl apply -f monitoring-test-apps.yaml

# Check all resources
kubectl get all --namespace=monitoring-workshop
```

### Step 3: Access Azure Monitor

Open Azure Monitor to view your cluster metrics:

```bash
# Open Azure Monitor for containers
az aks browse \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster"
```

### Step 4: Set Up Log Analytics Queries

Create useful Log Analytics queries:

```bash
# Get Log Analytics workspace ID
WORKSPACE_ID=$(az aks show \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --query "addonProfiles.omsagent.config.logAnalyticsWorkspaceResourceID" -o tsv)

# Query for container CPU usage
az monitor log-analytics query \
  --workspace "$WORKSPACE_ID" \
  --analytics-query "Perf | where ObjectName == 'K8SContainer' | where CounterName == 'cpuUsageNanoCores' | summarize avg(CounterValue) by bin(TimeGenerated, 1m), InstanceName | render timechart" \
  --output table

# Query for pod status
az monitor log-analytics query \
  --workspace "$WORKSPACE_ID" \
  --analytics-query "KubePodInventory | where TimeGenerated > ago(1h) | summarize count() by PodStatus, bin(TimeGenerated, 5m) | render timechart" \
  --output table
```

### Step 5: Create Custom Dashboards

Create a custom dashboard in Azure Monitor:

```bash
# Create dashboard JSON
cat <<EOF > custom-dashboard.json
{
  "lenses": {
    "0": {
      "order": 0,
      "parts": {
        "0": {
          "position": {
            "x": 0,
            "y": 0,
            "colSpan": 6,
            "rowSpan": 4
          },
          "metadata": {
            "inputs": [],
            "type": "Extension/Microsoft_OperationsManagementSuite_Workspace/PartType/LogsDashboardPart",
            "settings": {
              "content": {
                "Query": "Perf | where ObjectName == \"K8SContainer\" | where CounterName == \"cpuUsageNanoCores\" | summarize avg(CounterValue) by bin(TimeGenerated, 1m), InstanceName | render timechart",
                "PartTitle": "Container CPU Usage",
                "PartSubTitle": "Average CPU usage by container"
              }
            }
          }
        }
      }
    }
  },
  "name": "AKS Workshop Dashboard",
  "version": "1.0"
}
EOF

# Create the dashboard
az portal dashboard create \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-dashboard" \
  --location "Australia East" \
  --input-path custom-dashboard.json
```

### Step 6: Set Up Alerting

Create alert rules for monitoring:

```bash
# Create action group for notifications
az monitor action-group create \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-action-group" \
  --short-name "aks-alerts" \
  --action email "admin@example.com" "Admin Email"

# Create alert rule for high CPU usage
az monitor metrics alert create \
  --resource-group "rg-aks-workshop-01" \
  --name "high-cpu-alert" \
  --description "Alert when CPU usage is high" \
  --scopes "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-aks-workshop-01/providers/Microsoft.ContainerService/managedClusters/aks-workshop-01-cluster" \
  --condition "avg Percentage CPU > 80" \
  --window-size "5m" \
  --evaluation-frequency "1m" \
  --action "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-aks-workshop-01/providers/Microsoft.Insights/actionGroups/aks-workshop-action-group"
```

### Step 7: Install Prometheus and Grafana

Set up Prometheus and Grafana for advanced monitoring:

```bash
# Add Prometheus Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring-workshop \
  --create-namespace \
  --set grafana.enabled=true \
  --set prometheus.service.type=ClusterIP

# Check installation
kubectl get pods --namespace=monitoring-workshop
```

### Step 8: Monitor Application Performance

Generate load and monitor performance:

```bash
# Generate load on the high-memory-app
kubectl run load-generator \
  --image=busybox \
  --namespace=monitoring-workshop \
  --restart=Never \
  --rm -it \
  -- sh -c "while true; do wget -q -O- http://high-memory-service; sleep 0.1; done"

# Monitor resource usage
kubectl top pods --namespace=monitoring-workshop

# Monitor cluster events
kubectl get events --namespace=monitoring-workshop --sort-by='.lastTimestamp'
```

## Understanding Monitoring Concepts

### Azure Monitor Components

1. **Azure Monitor for Containers**
   - Collects metrics from AKS clusters
   - Provides pre-built dashboards
   - Integrates with Log Analytics

2. **Log Analytics Workspace**
   - Centralized log storage
   - Powerful query language (KQL)
   - Custom dashboards and alerts

### Monitoring Best Practices

1. **Resource Monitoring**
   - Monitor CPU, memory, and disk usage
   - Set appropriate alerts
   - Track resource trends

2. **Application Monitoring**
   - Monitor application logs
   - Track error rates
   - Monitor response times

## Hands-on Challenges

### Challenge 1: Create Custom Alerts
Create alerts for specific scenarios:

```bash
# Create alert for pod restart count
az monitor metrics alert create \
  --resource-group "rg-aks-workshop-01" \
  --name "pod-restart-alert" \
  --description "Alert when pods restart frequently" \
  --scopes "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-aks-workshop-01/providers/Microsoft.ContainerService/managedClusters/aks-workshop-01-cluster" \
  --condition "avg Restart Count > 5" \
  --window-size "10m" \
  --evaluation-frequency "2m"
```

### Challenge 2: Build Custom Dashboard
Create a comprehensive monitoring dashboard with multiple panels for CPU, memory, network, and application metrics.

## Troubleshooting

### Common Issues

1. **Monitoring Add-on Not Working**
   ```bash
   # Check omsagent pods
   kubectl get pods -n kube-system | grep omsagent
   
   # Check omsagent logs
   kubectl logs -n kube-system deployment/omsagent-rs
   ```

2. **Log Analytics Queries Not Working**
   ```bash
   # Verify workspace ID
   echo $WORKSPACE_ID
   
   # Test simple query
   az monitor log-analytics query \
     --workspace "$WORKSPACE_ID" \
     --analytics-query "KubePodInventory | limit 5" \
     --output table
   ```

## Cleanup

Clean up resources from this exercise:

```bash
# Delete all resources in the namespace
kubectl delete namespace monitoring-workshop

# Delete custom dashboard
az portal dashboard delete \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-dashboard"

# Delete alert rules
az monitor metrics alert delete \
  --resource-group "rg-aks-workshop-01" \
  --name "high-cpu-alert"

# Delete action group
az monitor action-group delete \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-action-group"

# Clean up temporary files
rm -f custom-dashboard.json
```

## Summary

In this exercise, you have:
- ✅ Set up Azure Monitor for containers
- ✅ Configured Log Analytics workspace
- ✅ Created custom metrics and dashboards
- ✅ Set up alerting and notifications
- ✅ Monitored cluster and application performance
- ✅ Analyzed logs and troubleshooted issues

## Next Steps

You're now ready to move to **Exercise 6: CI/CD with AKS** where you'll learn about GitHub Actions integration and automated deployments.

## Additional Resources

- [Azure Monitor for Containers](https://docs.microsoft.com/en-us/azure/azure-monitor/insights/container-insights-overview)
- [Log Analytics](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-overview)
- [Prometheus on AKS](https://docs.microsoft.com/en-us/azure/azure-monitor/insights/container-insights-prometheus)
- [KQL Query Language](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/) 