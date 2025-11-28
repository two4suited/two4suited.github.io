---
title: "Building a Modern Development Platform: Azure Infrastructure Foundation ☁️"
date: 2025-11-28T06:00:00-07:00
draft: false
categories: ["platform","azure","architecture","subscriptions","devops","hybrid-cloud"]
description: "Design and organization of a multi-subscription Azure platform - environment isolation, hybrid connectivity, shared services, and cross-cloud integration strategy"
---

[Series Posts](https://brianpsheridan.com/categories.html#platform)

## 🎯 Overview

A mature platform requires more than just throwing applications into Azure. You need thoughtful subscription organization that enables scaling, maintains security boundaries, and supports a hybrid and multi-cloud strategy. This post walks through how we organized six Azure subscriptions to support development teams, maintain infrastructure as code, integrate with on-premises systems, and connect to other cloud providers.

## 🏗️ Subscription Strategy: Six Subscriptions

Our platform uses six subscriptions, each with a specific purpose and isolation boundary:

```
┌──────────────────────────────────────────────────┐
│     Azure Subscription Structure (6 Subs)        │
├──────────────────────────────────────────────────┤
│                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────┐   │
│  │ TEST (CI)   │  │ QA (IT)     │  │ UAT     │   │
│  │             │  │             │  │ (Biz)   │   │
│  └─────────────┘  └─────────────┘  └─────────┘   │
│                                                  │
│  ┌─────────────┐  ┌─────────────────────────┐    │
│  │ PROD        │  │ Shared Services         │    │
│  │ (Live)      │  │ (ACR, Key Vault, etc)   │    │
│  └─────────────┘  └─────────────────────────┘    │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │ Networking & Hybrid Connectivity         │    │
│  │ (VPN, ExpressRoute, on-prem & clouds)    │    │
│  └──────────────────────────────────────────┘    │
│                                                  │
└──────────────────────────────────────────────────┘
```

### Subscription Breakdown

| Subscription | Purpose | Usage | Testing Focus |
|---|---|---|---|
| **Test** | CI/CD automated deployments | Code committed to repo → auto-deployed | Continuous integration, automated test suites |
| **QA** | IT quality assurance | Manual testing by QA teams | System functionality, compatibility, regression |
| **UAT** | Business user validation | Business users test features | Acceptance criteria, business workflows, sign-off |
| **Prod** | Production workloads | Live customer applications | Performance, stability, 24/7 support |
| **Shared Services** | Platform-wide resources | Centralized tooling, secrets, artifacts | Accessible to all environments |
| **Networking & Hybrid** | VPN, ExpressRoute, peering | Connect on-prem, AWS, GCP, other providers | Cross-environment connectivity |

## 🌍 Environment Subscriptions: The Four Application Platforms

Each of the four environment subscriptions (Test, QA, UAT, Prod) follows the same architecture pattern, with resources scaled and configured appropriately for that environment's purpose.

### ✨ Resources in Each Environment Subscription

Every environment subscription contains:

- **App Service Plans**: Each team manages their own App Service Plan for web apps and functions within predefined constraints (SKU, scaling limits, instance count)
- **Web Apps**: ASP.NET, Node.js, and other web application runtimes deployed to team-owned App Service Plans
- **Azure Functions**: Serverless compute for background jobs, event handlers, integrations on team-owned App Service Plans
- **API Management**: Central API gateway for version management, throttling, authentication
- **Front Door**: Global load balancing, DDoS protection, SSL/TLS termination
- **Storage Accounts**: Blob storage for artifacts, documents, backups; Queue storage for async messaging
- **Databases**: Azure SQL Database, Cosmos DB, or PostgreSQL for application data
- **Managed Identity & Service Principals**: Security credentials for app-to-Azure communication

### 🏢 Team-Managed App Service Plans

To balance autonomy with governance, each team manages their own App Service Plans within predefined boundaries:

**Constraints:**
- Maximum number of App Service Plans per team per environment
- Allowed SKU tiers (Standard, Premium, Isolated)
- Scaling limits (min/max instances)
- Cost allocation and monitoring requirements

**Benefits:**
- Teams have autonomy to scale and configure their own applications
- Prevents a single shared plan from becoming a bottleneck
- Isolates noisy neighbor scenarios (one team's traffic doesn't affect another)
- Clearer cost attribution per team/application
- Platform team maintains cost controls through predefined constraints

### 🔐 Credential Strategy: Two Service Principals Per Environment

Each environment subscription has **two service principals** to enforce principle of least privilege:

#### 1. **Control Plane Service Principal (Owner)**
- **Purpose**: Infrastructure deployments only (creating/modifying resources)
- **Permissions**: Owner role on the subscription
- **Usage**: Azure DevOps pipeline for infrastructure-as-code deployments (Terraform, Bicep)
- **Who Uses**: Infrastructure team, automation account
- **Example Tasks**: Create App Service Plans, provision databases, configure networking

#### 2. **Data Plane Service Principal (Application Deployment)**
- **Purpose**: Deploy applications and data (not infrastructure)
- **Permissions**: Contributor on specific resource groups, with custom roles for data-plane operations
- **Configured As**: Service Connection in Azure DevOps
- **Usage**: CI/CD pipelines for application deployments and data operations
- **Example Tasks**: Push code to Web Apps, copy files to storage, update app configuration, manage databases, insert/update data

**Data Plane Specific Permissions:**
- Deploy to App Services (publish profiles)
- Write to Storage Accounts (blobs, queues, tables)
- Execute stored procedures and manage database schema
- Update App Configuration values
- Create/update Azure SQL databases
- Manage Function App code and settings
- Read from Key Vault (retrieve connection strings)

**Example Permissions Separation:**

```
Control Plane (Owner):
✓ Create App Service Plans
✓ Create databases
✓ Modify network configuration
✓ Update resource definitions
✗ Deploy application code (cannot, doesn't need to)
✗ Manage database data (cannot)

Data Plane (Resource-specific permissions):
✓ Deploy code to Web Apps
✓ Execute database migrations
✓ Copy files to Storage Accounts
✓ Update app configuration values
✓ Insert/update database records
✓ Manage Azure Function code
✗ Create App Service Plans (cannot)
✗ Modify networking (cannot)
✗ Create new databases (cannot)
```

### 🔑 Application Authentication Strategy

Applications running in each environment use a **hybrid authentication approach**:

#### Managed Identity (Azure-to-Azure)
- **Used For**: Accessing other Azure resources
- **Examples**:
  - Web App → Azure SQL Database
  - Web App → Key Vault (fetch connection strings)
  - Web App → Storage Account (read/write blobs)
- **Benefit**: No credentials to manage, credentials never stored locally
- **How It Works**: App Service/Function App automatically gets an Azure AD identity that can be granted permissions to other resources

#### Service Principal (Application-Managed)
- **Used For**: When the application needs to authenticate to external systems or act on behalf of users
- **Examples**:
  - Calling downstream APIs that require service-to-service auth
  - Accessing on-premises resources through service principal
  - Multi-tenant scenarios where app needs to access customer resources
- **How It Works**: Credentials stored in Key Vault, app retrieves at runtime to authenticate

**Typical Flow:**

```
Web App Request
    ↓
App needs Azure SQL connection
    ↓
Managed Identity (automatic, no creds needed)
    ↓
Azure SQL grants access to Managed Identity
    ↓
Request succeeds, data returned
```

## 📦 Shared Services Subscription

The Shared Services subscription contains resources used across **all four environment subscriptions** and is not environment-specific.

### Resources in Shared Services

| Resource | Purpose | Used By |
|---|---|---|
| **Azure Container Registry (ACR)** | Centralized image repository | All environments pull container images |
| **App Configuration** | Centralized configuration management | All apps read config values |
| **Key Vault** | Secrets and certificate storage | All apps retrieve connection strings, API keys |
| **Storage Account (Terraform State)** | IaC state file storage | Terraform deployments from Azure DevOps |
| **Dynatrace Log Collector** | Centralized logging and APM | All applications send telemetry |
| **Other Shared Resources** | Network resources, diagnostic settings | All environments |

### 🏛️ Why Centralize These Resources?

✅ **Single Source of Truth**: One container registry means one place to manage images across all environments

✅ **Cost Efficiency**: Don't duplicate expensive resources (Key Vault, ACR) per environment

✅ **Consistency**: Same configurations, same secrets format, same logging approach across all teams

✅ **Access Control**: Grant dev/test/prod teams different levels of access to shared resources

✅ **Compliance**: Centralized audit logging and monitoring

## 🔗 Networking & Hybrid Connectivity Subscription

The sixth subscription handles all connectivity—both internal (between Azure subscriptions) and external (to on-premises, AWS, GCP, etc.).

### 🌉 Hybrid Connectivity Components

This subscription contains:

- **Azure VPN Gateway**: Encrypted connection to on-premises data centers
- **ExpressRoute (optional)**: Dedicated network connection for predictable latency/bandwidth
- **Hub Virtual Network**: Central networking hub
- **Spoke Virtual Networks**: One spoke per environment (peered to hub)
- **Network Peering**: Connections between hub and spoke networks
- **Firewall/NSGs**: Network segmentation and security policies
- **Routing Tables**: Directing traffic appropriately

### 🔄 Integration with Other Cloud Providers

Applications sometimes need to integrate with resources in:
- **AWS**: Lambda functions, DynamoDB, S3
- **GCP**: BigQuery, Cloud Storage, Compute Engine
- **On-Premises**: Legacy applications, databases, services

This is handled through:

1. **VPN/ExpressRoute to On-Premises**: Secure tunnel back to corporate network
2. **Inter-Cloud Networking**: Direct connections or VPN tunnels to AWS/GCP
3. **Hybrid Authentication**: Service principals bridging multiple clouds
4. **API Gateway Pattern**: Applications call APIs that abstract cloud boundaries

**Example Multi-Cloud Flow:**

```
Azure Web App
    ↓
Calls Azure API Management (gateway)
    ↓
Routes request through VPN to on-premises network
    ↓
Connects to on-premises core business system
    (legacy system handling key business functions)
    ↓
Core system processes request and returns data    
    ↓
Response back through VPN to Azure
    ↓
API Management returns data to web app
    ↓
Web app presents results to user
```



## 🚀 Deployment Pipelines: How Code Gets to Production

### Pipeline Architecture

```
Developer pushes code
    ↓
GitHub/Azure DevOps triggers build
    ↓
Tests run, artifacts created
    ↓
Artifact pushed to Shared Services ACR
    ↓
Deploy pipeline triggered
    ↓
Data Plane Service Principal authenticates
    ↓
Application code deployed to Web App/Function
    ↓
Managed Identity used for runtime Azure access
    ↓
Application running, serving requests
```

### Environment-Specific Pipeline Stages

All environments deploy through a **single pipeline** with environment-specific stages and approval gates:

- **Test Stage**: Automatically triggered on every code commit, deploys immediately (fast feedback for developers)
- **QA Stage**: Requires manual approval from QA lead, for IT QA team testing before business validation
- **UAT Stage**: Requires manual approval from business stakeholder, business users validate features and workflows, requires sign-off before prod
- **Prod Stage**: Requires manual approval from ops/platform team and business sign-off, production deployment with monitoring and notifications

**Pipeline Flow:**
```
Code Commit
    ↓
Test Stage (Auto-deploy)
    ↓
QA Stage (Manual Approval)
    ↓
UAT Stage (Manual Approval)
    ↓
Prod Stage (Manual Approval)
    ↓
Production Live
```

## 🔐 Security Boundaries & Isolation

### Subscription-Level Isolation

Each subscription is completely isolated:
- Resources in Test/QA/UAT cannot directly access Prod resources
- Earlier stage teams cannot accidentally modify Prod
- Cost tracking per subscription
- Security policies enforceable per subscription

### Role-Based Access Control (RBAC)

Permissions follow principle of least privilege:

| Team | Test | QA | UAT | Prod | Shared Services |
|---|---|---|---|---|---|
| **Developer** | Full | ✗ | ✗ | ✗ | Read-only |
| **QA/IT** | Read-only | Full | ✗ | ✗ | Read-only |
| **Business/UAT** | ✗ | ✗ | Full | ✗ | ✗ |
| **Ops/Platform** | Owner | Owner | Owner | Owner | Owner |
| **Control Plane SP** | Infra | Infra | Infra | Infra | — |
| **Data Plane SP** | Deploy | Deploy | Deploy | Deploy | — |

### Network Isolation

- **Network Policies**: NSGs prevent unauthorized traffic between subnets
- **Managed Identities**: No credentials stored, impossible to leak
- **Key Vault**: Secrets encrypted at rest and in transit
- **Service Endpoints**: Storage accounts only accessible from specific subnets

## 📊 Resource Naming Convention

Consistent naming across subscriptions enables automation and auditing. We use different patterns for different subscription types:

**Environment Subscriptions (Test, QA, UAT, Prod):**
```
Pattern: {business-unit}-{it-unit}-{app-name}-{environment}-{resource-type}-{number}

Examples:
fin-treasury-ledger-test-webapp-01
fin-treasury-ledger-test-sql-01
hr-benefits-payroll-qa-functions-batch
fin-treasury-ledger-prod-webapp-api
hr-benefits-payroll-prod-storage-data
```

**Shared Services Subscription:**
```
Pattern: {organization}-shared-{resource-type}

Examples:
acme-shared-acr
acme-shared-kv
acme-shared-appconfig
acme-shared-law
```

**Networking & Hybrid Subscription:**
```
Pattern: {organization}-network-{resource-type}-{region}

Examples:
acme-network-vnet-hub
acme-network-vnet-spoke-east
acme-network-vpn-onprem
```

Benefits:
- Business unit + IT unit + app name uniquely identifies each application project
- IT unit shows which team/department owns the application
- Environment clearly separated for resource filtering and automation
- Shared/Network prefixes identify cross-functional resources
- Easy to identify resource purpose, owner, and environment
- Monitoring alerts include full context for troubleshooting
- Cost allocation reports organized by business unit and IT unit
- Terraform can reference resources by predictable, hierarchical names

## 🎯 Best Practices We Follow

✅ **Subscription per Environment**: Prevents accidental prod changes from dev mistakes

✅ **Shared Services Isolation**: Separate subscription prevents environment-specific changes to shared resources

✅ **Two Service Principals**: Separates infrastructure concerns from application concerns

✅ **Managed Identity First**: Uses managed identity for all Azure-to-Azure communication

✅ **Service Principals in Key Vault**: Credentials never hardcoded, rotated centrally

✅ **Centralized Logging**: All environments send logs to shared Dynatrace

✅ **Hybrid Connectivity Isolated**: Dedicated subscription for networking prevents conflicts

✅ **Tagging Strategy**: All resources tagged with environment, owner, cost center for reporting

## ⚠️ Common Pitfalls We Avoided

❌ **Single Subscription for All Environments**: One mistake affects everything

❌ **Single Service Principal for Everything**: Can't restrict permissions effectively

❌ **Hardcoded Credentials in Code**: Credentials leak with source code

❌ **No Managed Identity**: Services store credentials locally, hard to rotate

❌ **Mixed Connectivity Concerns**: Networking conflicts between environments

❌ **No Central Logging**: Can't correlate issues across environments

## 🔍 Troubleshooting & Observability

### Multi-Environment Visibility

With centralized Dynatrace logging:
- Dev team can see their own logs
- Ops team can see all environments
- Alerts trigger based on cross-environment patterns
- Performance baselines compare across environments

### Common Troubleshooting Scenarios

**"Why can't my app access the database?"**
- Check: Does app's Managed Identity have database permissions?
- Check: Is database accessible from app's subnet?
- Check: Connection string correct in App Configuration?

**"Deployment failed to staging"**
- Check: Did Data Plane Service Principal authenticate?
- Check: Do permissions allow deployment to that App Service?
- Check: Is image in ACR accessible to staging subscription?

**"On-premises system can't reach Azure app"**
- Check: Is VPN connection active?
- Check: Do routing tables direct traffic correctly?
- Check: Do NSGs allow inbound traffic from on-prem CIDR?

## 📈 Scaling Considerations

This architecture scales to support:
- **Multiple teams**: Each team gets isolated environments but shared platform services
- **Multiple regions**: Replicate subscription structure across regions
- **Multi-cloud**: Networking subscription acts as hub for AWS, GCP integration
- **Compliance**: Audit subscription-level activity, enforce policies per environment

## 🤔 Questions to Consider for Your Platform

- Do you need a dedicated integration/staging environment before production?
- Will you support multiple regions, and if so, do you need subscriptions per region?
- Do you need tighter RBAC controls for certain teams?
- How will you handle disaster recovery across subscriptions?
- Do you need to integrate with other cloud providers or on-premises systems now or in the future?

---

This subscription and authentication strategy enables teams to move fast in development, maintain stability in production, and keep infrastructure secure through proper isolation and credential management.

**Series Links**: [Building a Modern Development Platform Home](https://brianpsheridan.com/categories.html#platform)
