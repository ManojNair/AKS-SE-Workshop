# Exercise 10: Service Mesh - Managed Istio Add-on

## Overview

This exercise covers the implementation of Service Mesh using the Managed Istio Add-on in Azure Kubernetes Service (AKS). You'll learn how to set up advanced networking, security, and observability features for microservices communication.

## Learning Objectives

- Understand Service Mesh concepts and Istio architecture
- Enable and configure Managed Istio Add-on in AKS
- Implement traffic management and routing
- Set up security policies and mTLS
- Configure observability and monitoring
- Implement circuit breakers and fault injection
- Create canary deployments with Istio

## Prerequisites

- Completed Exercises 1-9
- Azure subscription with appropriate permissions
- Azure CLI installed and configured
- kubectl installed
- Understanding of microservices architecture
- Basic knowledge of Kubernetes networking

## Duration

- **Estimated Time**: 2-3 hours
- **Difficulty**: Advanced

---

## Part 1: Understanding Service Mesh and Istio

### 1.1 What is Service Mesh?

Service Mesh is a dedicated infrastructure layer that handles service-to-service communication in a microservices architecture. It provides:

**Key Features:**
- **Traffic Management**: Load balancing, routing, and traffic splitting
- **Security**: mTLS encryption, authentication, and authorization
- **Observability**: Metrics, logging, and distributed tracing
- **Resilience**: Circuit breakers, retries, and fault injection
- **Policy Enforcement**: Rate limiting and access control

### 1.2 Istio Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Application   │    │   Application   │    │   Application   │
│   Container     │    │   Container     │    │   Container     │
├─────────────────┤    ├─────────────────┤    ├─────────────────┤
│   Sidecar       │    │   Sidecar       │    │   Sidecar       │
│   Proxy         │    │   Proxy         │    │   Proxy         │
│   (Envoy)       │    │   (Envoy)       │    │   (Envoy)       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────────┐
                    │   Control Plane │
                    │   (Istiod)      │
                    └─────────────────┘
```

### 1.3 Managed Istio Add-on Benefits

- **Fully Managed**: Azure handles Istio control plane updates
- **Integrated**: Seamless integration with AKS
- **Simplified**: Reduced operational overhead
- **Enterprise Ready**: Production-grade security and reliability

---

## Part 2: Enable Managed Istio Add-on

### 2.1 Check Istio Add-on Availability

```bash
# Check if Istio add-on is available in your region
az aks get-versions \
  --location australiaeast \
  --query "orchestrators[?orchestratorVersion=='1.28.0'].addonProfiles.istio" \
  --output table

# Check current AKS cluster version
az aks show \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --query "kubernetesVersion"
```

**Windows PowerShell:**
```powershell
# Check if Istio add-on is available in your region
az aks get-versions `
  --location australiaeast `
  --query "orchestrators[?orchestratorVersion=='1.28.0'].addonProfiles.istio" `
  --output table

# Check current AKS cluster version
az aks show `
  --resource-group "rg-aks-workshop-01" `
  --name "aks-workshop-01-cluster" `
  --query "kubernetesVersion"
```

### 2.2 Enable Istio Add-on

```bash
# Enable Istio add-on
az aks enable-addons \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --addons istio

# Verify Istio installation
kubectl get pods -n aks-istio-system
kubectl get svc -n aks-istio-system

# Check Istio components
kubectl get pods -n aks-istio-system -l app=istiod
kubectl get pods -n aks-istio-system -l app=istio-ingressgateway
```

**Windows PowerShell:**
```powershell
# Enable Istio add-on
az aks enable-addons `
  --resource-group "rg-aks-workshop-01" `
  --name "aks-workshop-01-cluster" `
  --addons istio

# Verify Istio installation
kubectl get pods -n aks-istio-system
kubectl get svc -n aks-istio-system

# Check Istio components
kubectl get pods -n aks-istio-system -l app=istiod
kubectl get pods -n aks-istio-system -l app=istio-ingressgateway
```

### 2.3 Configure Istio Settings

```bash
# Create Istio configuration
cat <<EOF | kubectl apply -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: istio-config
spec:
  profile: default
  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: 500m
            memory: 2048Mi
          limits:
            cpu: 1000m
            memory: 4096Mi
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 2000m
            memory: 1024Mi
  values:
    global:
      proxy:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
    pilot:
      autoscaleEnabled: true
      autoscaleMin: 1
      autoscaleMax: 3
      resources:
        requests:
          cpu: 500m
          memory: 2048Mi
        limits:
          cpu: 1000m
          memory: 4096Mi
EOF
```

---

## Part 3: Deploy Sample Applications

### 3.1 Create Namespace and Enable Istio

```bash
# Create namespace for sample applications
kubectl create namespace istio-demo

# Enable Istio sidecar injection
kubectl label namespace istio-demo istio-injection=enabled

# Verify namespace labeling
kubectl get namespace istio-demo --show-labels
```

### 3.2 Deploy Bookinfo Application

```bash
# Deploy Bookinfo application
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/bookinfo/platform/kube/bookinfo.yaml -n istio-demo

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod -l app=productpage -n istio-demo --timeout=300s
kubectl wait --for=condition=ready pod -l app=reviews -n istio-demo --timeout=300s
kubectl wait --for=condition=ready pod -l app=ratings -n istio-demo --timeout=300s
kubectl wait --for=condition=ready pod -l app=details -n istio-demo --timeout=300s

# Verify deployment
kubectl get pods -n istio-demo
kubectl get svc -n istio-demo
```

### 3.3 Create Istio Gateway and Virtual Service

```bash
# Create Istio Gateway
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: istio-demo
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
EOF

# Create Virtual Service
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo
  namespace: istio-demo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        prefix: /productpage
    - uri:
        prefix: /static
    - uri:
        prefix: /login
    - uri:
        prefix: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
EOF

# Get ingress gateway external IP
kubectl get svc istio-ingressgateway -n aks-istio-system
```

---

## Part 4: Traffic Management

### 4.1 Implement Traffic Splitting

```bash
# Create Destination Rules
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: productpage
  namespace: istio-demo
spec:
  host: productpage
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
  namespace: istio-demo
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ratings
  namespace: istio-demo
spec:
  host: ratings
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: details
  namespace: istio-demo
spec:
  host: details
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF

# Update Virtual Service for traffic splitting
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 70
    - destination:
        host: reviews
        subset: v2
      weight: 20
    - destination:
        host: reviews
        subset: v3
      weight: 10
EOF
```

### 4.2 Implement Canary Deployment

```bash
# Deploy new version of reviews service
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/bookinfo/platform/kube/bookinfo.yaml -n istio-demo

# Create canary deployment with traffic shifting
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-canary
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
EOF

# Gradually shift traffic (simulate canary deployment)
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-canary
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v2
      weight: 50
EOF
```

---

## Part 5: Security Implementation

### 5.1 Enable mTLS

```bash
# Create PeerAuthentication policy
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-demo
spec:
  mtls:
    mode: STRICT
EOF

# Create AuthorizationPolicy
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: productpage-policy
  namespace: istio-demo
spec:
  selector:
    matchLabels:
      app: productpage
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/istio-demo/sa/bookinfo-productpage"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/api/v1/products/*"]
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: reviews-policy
  namespace: istio-demo
spec:
  selector:
    matchLabels:
      app: reviews
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/istio-demo/sa/bookinfo-productpage"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/reviews/*"]
EOF
```

### 5.2 Configure Service Account

```bash
# Create service account for Bookinfo
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-productpage
  namespace: istio-demo
  labels:
    account: productpage
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-reviews
  namespace: istio-demo
  labels:
    account: reviews
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-ratings
  namespace: istio-demo
  labels:
    account: ratings
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-details
  namespace: istio-demo
  labels:
    account: details
EOF

# Update deployments to use service accounts
kubectl patch deployment productpage-v1 -n istio-demo -p '{"spec":{"template":{"spec":{"serviceAccountName":"bookinfo-productpage"}}}}}'
kubectl patch deployment reviews-v1 -n istio-demo -p '{"spec":{"template":{"spec":{"serviceAccountName":"bookinfo-reviews"}}}}}'
kubectl patch deployment reviews-v2 -n istio-demo -p '{"spec":{"template":{"spec":{"serviceAccountName":"bookinfo-reviews"}}}}}'
kubectl patch deployment reviews-v3 -n istio-demo -p '{"spec":{"template":{"spec":{"serviceAccountName":"bookinfo-reviews"}}}}}'
kubectl patch deployment ratings-v1 -n istio-demo -p '{"spec":{"template":{"spec":{"serviceAccountName":"bookinfo-ratings"}}}}}'
kubectl patch deployment details-v1 -n istio-demo -p '{"spec":{"template":{"spec":{"serviceAccountName":"bookinfo-details"}}}}}'
```

---

## Part 6: Observability and Monitoring

### 6.1 Enable Istio Telemetry

```bash
# Create Telemetry configuration
cat <<EOF | kubectl apply -f -
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-demo
spec:
  tracing:
  - randomSamplingPercentage: 100.0
  metrics:
  - providers:
    - name: prometheus
  accessLogging:
  - providers:
    - name: envoy
EOF
```

### 6.2 Set Up Kiali Dashboard

```bash
# Install Kiali
helm repo add kiali https://kiali.org/helm-charts
helm repo update

# Install Kiali with Istio integration
helm install kiali-server kiali/kiali \
  --namespace aks-istio-system \
  --set auth.strategy=anonymous \
  --set external_services.istio.root_namespace=aks-istio-system \
  --set external_services.istio.url=http://istiod.aks-istio-system.svc:15012 \
  --set external_services.prometheus.url=http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090

# Port forward to access Kiali
kubectl port-forward -n aks-istio-system svc/kiali-server 20001:20001
```

### 6.3 Configure Prometheus and Grafana

```bash
# Install Prometheus and Grafana
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.enabled=true \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false

# Configure Istio metrics collection
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-component-monitor
  namespace: monitoring
  labels:
    monitoring: istio-components
spec:
  selector:
    matchLabels:
      istio: mixer
  namespaceSelector:
    matchNames:
    - aks-istio-system
  endpoints:
  - port: prometheus
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: envoy-stats-monitor
  namespace: monitoring
  labels:
    monitoring: istio-proxies
spec:
  selector:
    matchLabels:
      istio-mixer-type: telemetry
  namespaceSelector:
    matchNames:
    - istio-demo
  endpoints:
  - path: /stats/prometheus
    port: http-envoy-prom
EOF
```

---

## Part 7: Circuit Breakers and Fault Injection

### 7.1 Implement Circuit Breaker

```bash
# Create circuit breaker for ratings service
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ratings-circuit-breaker
  namespace: istio-demo
spec:
  host: ratings
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
  subsets:
  - name: v1
    labels:
      version: v1
EOF
```

### 7.2 Fault Injection Testing

```bash
# Create fault injection for testing
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings-fault-injection
  namespace: istio-demo
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 10
        fixedDelay: 5s
      abort:
        percentage:
          value: 10
        httpStatus: 500
    route:
    - destination:
        host: ratings
        subset: v1
EOF

# Test fault injection
for i in {1..10}; do
  curl -s http://$(kubectl get svc istio-ingressgateway -n aks-istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')/productpage | grep -o "reviews-v[0-9]" | head -1
  sleep 1
done
```

---

## Part 8: Advanced Traffic Management

### 8.1 Request Routing Based on Headers

```bash
# Create header-based routing
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-header-routing
  namespace: istio-demo
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
EOF
```

### 8.2 Retry and Timeout Policies

```bash
# Create retry and timeout policies
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: productpage-retry-timeout
  namespace: istio-demo
spec:
  hosts:
  - productpage
  http:
  - route:
    - destination:
        host: productpage
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: connect-failure,refused-stream,unavailable,cancelled,retriable-status-codes
    timeout: 10s
EOF
```

---

## Part 9: Monitoring and Troubleshooting

### 9.1 Istio Metrics and Logs

```bash
# Check Istio metrics
kubectl exec -n istio-demo $(kubectl get pod -n istio-demo -l app=productpage -o jsonpath='{.items[0].metadata.name}') -c istio-proxy -- curl -s http://localhost:15090/stats/prometheus | grep istio

# Check Istio proxy logs
kubectl logs -n istio-demo $(kubectl get pod -n istio-demo -l app=productpage -o jsonpath='{.items[0].metadata.name}') -c istio-proxy

# Check Istio control plane logs
kubectl logs -n aks-istio-system -l app=istiod -c discovery

# Check Istio ingress gateway logs
kubectl logs -n aks-istio-system -l app=istio-ingressgateway -c istio-proxy
```

### 9.2 Common Issues and Solutions

**Sidecar Injection Issues:**
```bash
# Check if sidecar injection is working
kubectl get pods -n istio-demo -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'

# Manually inject sidecar if needed
kubectl patch deployment productpage-v1 -n istio-demo -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}'
```

**Traffic Routing Issues:**
```bash
# Check Virtual Service configuration
kubectl get virtualservice -n istio-demo -o yaml

# Check Destination Rules
kubectl get destinationrule -n istio-demo -o yaml

# Verify service endpoints
kubectl get endpoints -n istio-demo
```

**Security Policy Issues:**
```bash
# Check PeerAuthentication
kubectl get peerauthentication -n istio-demo -o yaml

# Check AuthorizationPolicy
kubectl get authorizationpolicy -n istio-demo -o yaml

# Verify mTLS status
kubectl exec -n istio-demo $(kubectl get pod -n istio-demo -l app=productpage -o jsonpath='{.items[0].metadata.name}') -c istio-proxy -- istioctl authn tls-check productpage.istio-demo.svc.cluster.local
```

---

## Challenges and Exercises

### Challenge 1: Multi-Version Deployment
Create a blue-green deployment strategy using Istio traffic management.

### Challenge 2: Security Hardening
Implement comprehensive security policies including authentication and authorization.

### Challenge 3: Observability Dashboard
Create custom Grafana dashboards for Istio metrics and traces.

### Challenge 4: Performance Testing
Set up load testing with circuit breakers and fault injection.

---

## Cleanup

```bash
# Delete sample applications
kubectl delete namespace istio-demo

# Uninstall Kiali
helm uninstall kiali-server -n aks-istio-system

# Uninstall Prometheus and Grafana
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring

# Disable Istio add-on
az aks disable-addons \
  --resource-group "rg-aks-workshop-01" \
  --name "aks-workshop-01-cluster" \
  --addons istio

# Verify Istio removal
kubectl get pods -n aks-istio-system
```

**Windows PowerShell:**
```powershell
# Delete sample applications
kubectl delete namespace istio-demo

# Uninstall Kiali
helm uninstall kiali-server -n aks-istio-system

# Uninstall Prometheus and Grafana
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring

# Disable Istio add-on
az aks disable-addons `
  --resource-group "rg-aks-workshop-01" `
  --name "aks-workshop-01-cluster" `
  --addons istio

# Verify Istio removal
kubectl get pods -n aks-istio-system
```

---

## Summary

In this exercise, you learned:

✅ **Service Mesh Concepts**: Understanding Istio architecture and benefits  
✅ **Managed Istio Setup**: Enabling and configuring Istio add-on in AKS  
✅ **Traffic Management**: Implementing routing, splitting, and canary deployments  
✅ **Security Implementation**: Setting up mTLS, authentication, and authorization  
✅ **Observability**: Configuring monitoring, tracing, and dashboards  
✅ **Resilience**: Implementing circuit breakers and fault injection  
✅ **Advanced Features**: Header-based routing and retry policies  
✅ **Troubleshooting**: Common issues and monitoring techniques  

## Next Steps

- Implement Istio in production environments
- Set up advanced security policies
- Create custom monitoring dashboards
- Integrate with existing CI/CD pipelines
- Develop custom Istio resources for specific use cases

## Additional Resources

- [Istio Documentation](https://istio.io/docs/)
- [AKS Istio Add-on](https://docs.microsoft.com/en-us/azure/aks/istio-about)
- [Istio Security](https://istio.io/docs/concepts/security/)
- [Istio Traffic Management](https://istio.io/docs/concepts/traffic-management/)
- [Istio Observability](https://istio.io/docs/concepts/observability/)

---

**Congratulations!** You've completed the Service Mesh exercise. You now have comprehensive knowledge of implementing Istio in AKS for advanced microservices networking, security, and observability.
</rewritten_file> 