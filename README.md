# Azure Kubernetes Service (AKS) Workshop

Welcome to the comprehensive AKS Workshop! This workshop is designed to take you from beginner to advanced AKS concepts through hands-on exercises.

> **📖 This is the main README for the AKS Workshop. Start here to get an overview and navigate to specific exercises.**

## 🎯 Workshop Overview

This comprehensive workshop consists of **11 progressive exercises** that build your knowledge and skills with Azure Kubernetes Service (AKS). Each exercise focuses on specific aspects of AKS management, deployment, and operations, taking you from basic cluster setup to advanced service mesh implementation.

### 📊 Workshop Statistics
- **Total Exercises**: 11
- **Total Duration**: 8-12 hours
- **Difficulty Levels**: Beginner → Advanced
- **Cross-Platform Support**: Windows, macOS, Linux
- **Azure Region**: Australia East (optimized for APAC)
- **Security Focus**: OIDC, mTLS, RBAC, and best practices

## Prerequisites

Before starting this workshop, ensure you have:

- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) (version 2.0.79 or later)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) (Kubernetes command-line tool)
- [Docker](https://docs.docker.com/get-docker/) (optional, for local container development)
- Active Azure subscription with sufficient credits
- Basic understanding of Kubernetes concepts

### Installation Instructions by Platform

#### Windows
```powershell
# Install Azure CLI via winget
winget install Microsoft.AzureCLI

# Install kubectl via winget
winget install Kubernetes.kubectl

# Install Docker Desktop for Windows
# Download from: https://docs.docker.com/desktop/install/windows-install/
```

#### macOS
```bash
# Install Azure CLI via Homebrew
brew install azure-cli

# Install kubectl via Homebrew
brew install kubectl

# Install Docker Desktop for Mac
# Download from: https://docs.docker.com/desktop/install/mac-install/
```

#### Linux (Ubuntu/Debian)
```bash
# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Install kubectl
sudo az aks install-cli

# Install Docker
sudo apt-get update
sudo apt-get install docker.io
sudo systemctl start docker
sudo systemctl enable docker
```

## Workshop Structure

### 📋 Table of Contents

| Exercise | Topic | File | Duration | Difficulty |
|----------|-------|------|----------|------------|
| **1** | AKS Cluster Fundamentals | [exercise-01-aks-cluster-fundamentals.md](./exercise-01-aks-cluster-fundamentals.md) | 30-45 min | Beginner |
| **2** | Application Deployment | [exercise-02-application-deployment.md](./exercise-02-application-deployment.md) | 45-60 min | Beginner |
| **3** | Scaling & Resource Management | [exercise-03-scaling-resource-management.md](./exercise-03-scaling-resource-management.md) | 30-45 min | Intermediate |
| **4** | Networking & Security | [exercise-04-networking-security.md](./exercise-04-networking-security.md) | 45-60 min | Intermediate |
| **5** | Monitoring & Logging | [exercise-05-monitoring-logging.md](./exercise-05-monitoring-logging.md) | 30-45 min | Intermediate |
| **6** | CI/CD with AKS (OIDC) | [exercise-06-cicd-aks.md](./exercise-06-cicd-aks.md) | 45-60 min | Intermediate |
| **7** | Advanced AKS Features | [exercise-07-advanced-aks-features.md](./exercise-07-advanced-aks-features.md) | 45-60 min | Advanced |
| **8** | Disaster Recovery & Backup | [exercise-08-disaster-recovery-backup.md](./exercise-08-disaster-recovery-backup.md) | 30-45 min | Advanced |
| **9** | Node Auto Provisioner (Karpenter) | [exercise-09-node-auto-provisioner-karpenter.md](./exercise-09-node-auto-provisioner-karpenter.md) | 45-60 min | Advanced |
| **10** | Service Mesh (Managed Istio) | [exercise-10-service-mesh-managed-istio.md](./exercise-10-service-mesh-managed-istio.md) | 45-60 min | Advanced |
| **11** | Containerization Assessment | [exercise-11-containerization-candidate-identification.md](./exercise-11-containerization-candidate-identification.md) | 30-45 min | Intermediate |

### 🎯 Exercise Overview

#### **Exercise 1: AKS Cluster Fundamentals**
- Create your first AKS cluster
- Understand AKS architecture and components
- Basic cluster management and operations
- Cross-platform setup (Windows, macOS, Linux)

#### **Exercise 2: Application Deployment and Management**
- Deploy applications to AKS using various methods
- Work with Kubernetes manifests and YAML
- Service types and networking concepts
- Multi-container applications

#### **Exercise 3: Scaling and Resource Management**
- Horizontal and vertical pod scaling
- Resource quotas, limits, and requests
- Cluster autoscaler configuration
- Node pool management

#### **Exercise 4: Networking and Security**
- Network policies and security groups
- Azure CNI vs Kubenet networking
- RBAC and security best practices
- Pod security policies

#### **Exercise 5: Monitoring and Logging**
- Azure Monitor for containers setup
- Log Analytics workspace configuration
- Custom metrics and alerting
- Prometheus and Grafana integration

#### **Exercise 6: CI/CD with AKS (OIDC)**
- GitHub Actions integration with OpenID Connect
- Azure Container Registry integration
- Automated deployment pipelines
- Security-hardened authentication

#### **Exercise 7: Advanced AKS Features**
- Virtual nodes and serverless containers
- GPU workloads and machine learning
- Windows containers support
- Advanced networking features

#### **Exercise 8: Disaster Recovery and Backup**
- Multi-region deployment strategies
- Backup and restore procedures
- Business continuity planning
- Cross-region failover testing

#### **Exercise 9: Node Auto Provisioner (Karpenter)**
- Dynamic node provisioning
- Cost optimization with spot instances
- Multi-zone distribution
- Advanced scaling strategies

#### **Exercise 10: Service Mesh (Managed Istio)**
- Istio architecture and concepts
- Traffic management and routing
- Security policies and mTLS
- Observability and monitoring

#### **Exercise 11: Containerization Assessment**
- Identifying containerization candidates
- Azure tools for assessment
- Migration planning and strategies
- Best practices for modernization

## Learning Objectives

By the end of this workshop, you will be able to:

1. **Create and manage AKS clusters** with different configurations and architectures
2. **Deploy and manage applications** using various Kubernetes resources and patterns
3. **Implement scaling strategies** for optimal resource utilization and cost management
4. **Configure networking and security** for production workloads with best practices
5. **Set up monitoring and logging** for operational visibility and troubleshooting
6. **Implement CI/CD pipelines** with secure authentication using OpenID Connect
7. **Utilize advanced AKS features** for specialized workloads and edge scenarios
8. **Plan and implement disaster recovery** strategies for business continuity
9. **Configure dynamic node provisioning** with Karpenter for cost optimization
10. **Implement service mesh** with Istio for advanced microservices management
11. **Assess and plan containerization** strategies using Azure tools and best practices

## Workshop Environment

### Azure Region
All exercises use the **Australia East** region for consistency and optimal performance in the APAC region.

### Resource Naming Convention
- Resource Groups: `rg-aks-workshop-{exercise-number}`
- AKS Clusters: `aks-workshop-{exercise-number}-cluster`
- Virtual Networks: `vnet-aks-workshop-{exercise-number}`
- Storage Accounts: `staks{exercise-number}{random}`
- Key Vaults: `kv-aks-workshop-{exercise-number}`

### Cost Management
- All resources are tagged with `Environment=Workshop` and `Purpose=Learning`
- Exercises include cleanup instructions to avoid unnecessary costs
- Use of cost-effective VM sizes for development/learning

## Getting Started

1. **Complete the prerequisites** installation for your platform
2. **Authenticate with Azure**:
   ```bash
   az login
   az account set --subscription "Your-Subscription-Name-or-ID"
   ```
3. **Start with Exercise 1** and progress through each exercise sequentially
4. **Complete the cleanup** steps at the end of each exercise

## Support and Resources

### Documentation
- [AKS Documentation](https://docs.microsoft.com/en-us/azure/aks/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Azure CLI Documentation](https://docs.microsoft.com/en-us/cli/azure/)

### Community Support
- [Azure Community](https://techcommunity.microsoft.com/t5/azure/ct-p/Azure)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/azure-aks)
- [GitHub Issues](https://github.com/Azure/AKS/issues)

## Workshop Completion

After completing all 11 exercises, you will have:
- **Comprehensive hands-on experience** with AKS cluster management and operations
- **Practical knowledge** of Kubernetes concepts and advanced features
- **Deep understanding** of Azure-specific AKS capabilities and integrations
- **Production-ready skills** for implementing AKS in enterprise environments
- **Modern DevOps practices** including secure CI/CD and service mesh implementation
- **Cost optimization strategies** using dynamic provisioning and spot instances
- **Disaster recovery planning** and implementation capabilities
- **Containerization assessment** and migration planning expertise

## Next Steps

After completing this workshop, consider:
- Taking the [AZ-104: Microsoft Azure Administrator](https://docs.microsoft.com/en-us/certifications/exams/az-104/) certification
- Exploring [AKS Best Practices](https://docs.microsoft.com/en-us/azure/aks/best-practices)
- Joining the [Azure Kubernetes Service Community](https://github.com/Azure/AKS)

---

**Note**: This workshop is designed for learning purposes. Always follow your organization's security and compliance policies when implementing these concepts in production environments. 