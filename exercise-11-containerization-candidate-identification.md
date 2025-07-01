# Exercise 11: Identifying the Right Candidate for Containerization (Azure Focus)

## Overview

This exercise provides detailed guidance on how to identify the right applications and workloads for containerization, and introduces tools and services in Microsoft Azure that can help with this process.

## Learning Objectives

- Understand the criteria for selecting containerization candidates
- Evaluate legacy and modern workloads for containerization
- Use Azure tools to assess and plan containerization
- Learn best practices for containerization readiness

## Prerequisites

- Basic understanding of application architectures (monolith, microservices)
- Access to Azure Portal and Azure CLI

## Duration

- **Estimated Time**: 1-2 hours
- **Difficulty**: Intermediate

---

## Part 1: Why Containerize?

- **Portability**: Run anywhere (on-prem, cloud, hybrid)
- **Scalability**: Easily scale workloads
- **DevOps Enablement**: Streamline CI/CD
- **Resource Efficiency**: Better utilization than VMs
- **Isolation**: Improved security and reliability

---

## Part 2: Criteria for Containerization Candidates

### 2.1 Good Candidates
- **Stateless applications** (web APIs, microservices)
- **12-factor apps**
- **Modern web apps**
- **Batch jobs and scheduled tasks**
- **Apps with clear dependencies**
- **Apps already running on Linux or Windows Server 2016+**

### 2.2 Challenging Candidates
- **Stateful monoliths** with tight coupling
- **Apps with heavy GUI dependencies**
- **Apps requiring legacy OS or hardware**
- **Apps with complex licensing**
- **Apps with high I/O or low-latency requirements**

### 2.3 Assessment Checklist
- Is the app stateless or can it be made stateless?
- Can persistent data be externalized (DB, storage)?
- Are dependencies well-documented?
- Is the app actively maintained?
- Is there a business driver for modernization?

---

## Part 3: Azure Tools for Containerization Assessment

### 3.1 Azure Migrate: App Containerization Tool

Azure Migrate provides a dedicated tool for assessing and containerizing .NET and Java web apps:

- **Discover**: Scan on-premises or VM-based apps
- **Assess**: Identify containerization readiness
- **Containerize**: Generate Dockerfiles and Kubernetes manifests

**Steps:**
1. Go to Azure Portal → Azure Migrate
2. Add a new project and select "App Containerization"
3. Download and run the Azure Migrate appliance on your source environment
4. Review assessment reports for containerization suitability
5. Use the tool to generate container artifacts

**References:**
- [Azure Migrate: App Containerization](https://learn.microsoft.com/en-us/azure/migrate/tutorial-app-containerization/)

### 3.2 App Service Migration Assistant

- Assesses web apps for migration to Azure App Service or containers
- Provides compatibility reports and migration guidance

**Reference:**
- [App Service Migration Assistant](https://appmigration.microsoft.com/)

### 3.3 Azure Container Apps Assessment

- For microservices and event-driven workloads
- Use Azure Container Apps for serverless containers
- Assess suitability based on statelessness and event-driven patterns

### 3.4 Azure Advisor & Workbooks

- Use Azure Advisor for modernization recommendations
- Use Azure Workbooks for custom assessment dashboards

---

## Part 4: Hands-on - Assessing a Sample Application

### 4.1 Using Azure Migrate for Assessment

```bash
# Prerequisite: Azure CLI installed and logged in
az extension add --name migrate

# Create Azure Migrate project
az migrate project create \
  --resource-group myResourceGroup \
  --name myMigrateProject \
  --location australiaeast

# Register the appliance and start discovery (see Azure Portal for details)
```

### 4.2 Reviewing Assessment Results
- Review the Azure Migrate dashboard for containerization readiness
- Download generated Dockerfiles and manifests
- Identify blockers and remediation steps

---

## Part 5: Best Practices for Containerization Readiness

- **Refactor for statelessness** where possible
- **Externalize configuration** (use environment variables, Azure Key Vault)
- **Automate builds** with Dockerfiles and CI/CD
- **Document dependencies** and required ports
- **Test locally** before cloud migration
- **Use Azure Container Registry** for image storage

---

## Challenges

1. **Assessment Challenge**: Use Azure Migrate to assess a legacy .NET or Java app for containerization.
2. **Modernization Plan**: Create a modernization plan for a monolithic app.
3. **Tool Comparison**: Compare Azure Migrate with other tools (e.g., App Service Migration Assistant).

---

## Cleanup

- Remove Azure Migrate projects and resources if not needed
- Delete any test container images from Azure Container Registry

---

## Summary

In this exercise, you learned:
- How to identify the right candidates for containerization
- What makes an app a good or poor candidate
- How to use Azure Migrate and other Azure tools for assessment
- Best practices for preparing apps for containers

## Additional Resources

- [Azure Migrate Documentation](https://learn.microsoft.com/en-us/azure/migrate/)
- [Azure Containerization Guidance](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/modernize-apps)
- [App Service Migration Assistant](https://appmigration.microsoft.com/)
- [Azure Container Registry](https://learn.microsoft.com/en-us/azure/container-registry/)

---

**Congratulations!** You now know how to identify and assess applications for containerization using Microsoft Azure. 