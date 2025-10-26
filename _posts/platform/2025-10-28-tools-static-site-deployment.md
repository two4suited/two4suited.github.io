---
title: "Building a Modern Development Platform: Deploying Platform Documentation with Azure Storage and Front Door 📚"
date: 2025-10-26T06:00:00-07:00
draft: false
categories: ["platform","azure","terraform","static-sites","documentation","front-door"]
description: "Deploying platform documentation sites to Azure Storage with Front Door CDN using TechDocs, Azure DevOps pipelines, and Terraform Cloud for infrastructure as code"
---

[Series Posts](https://brianpsheridan.com/categories.html#platform)

## Introduction

A modern development platform needs excellent documentation. But documentation infrastructure shouldn't be expensive or complex to maintain. In this post, we'll deploy a static documentation site to Azure Storage with Azure Front Door for global distribution - all managed through Terraform Cloud and automated with Azure DevOps pipelines.

This approach gives us:
- **Low Cost**: Azure Storage static websites cost pennies per month
- **Global Performance**: Front Door CDN distributes content worldwide
- **Automated Deployment**: Azure DevOps pipelines rebuild and deploy on every commit
- **Infrastructure as Code**: Terraform Cloud manages all Azure resources
- **Developer-Friendly**: TechDocs generates beautiful documentation from Markdown

The code for this post is available in the [blog-platform-aspire repository](https://github.com/two4suited/blog-platform-aspire/tree/aspire-tools-static-site).

## Table of Contents

- [Why Static Site Hosting on Azure Storage?](#why-static-site-hosting-on-azure-storage)
- [Architecture Overview](#architecture-overview)
- [What is TechDocs?](#what-is-techdocs)
- [Setting Up Terraform Cloud](#setting-up-terraform-cloud)
- [Azure Infrastructure with Terraform](#azure-infrastructure-with-terraform)
- [Setting Up TechDocs](#setting-up-techdocs)
- [Setting Up Azure DevOps](#setting-up-azure-devops)
- [Azure DevOps Pipeline](#azure-devops-pipeline)
- [Deployment Workflow](#deployment-workflow)
- [Monitoring and Analytics](#monitoring-and-analytics)
- [Testing the Deployment](#testing-the-deployment)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)
- [Cost Optimization](#cost-optimization)
- [Cleaning Up Resources](#cleaning-up-resources)

## Why Static Site Hosting on Azure Storage? 💾

Traditional web hosting requires web servers, compute resources, and ongoing maintenance. Static site hosting on Azure Storage eliminates all of that:

**Cost Comparison:** 💰
- App Service Basic (B1): ~$13/month
- Azure Storage static website: ~$0.50/month
- Savings: 96% reduction

**What You Get:** ✨
- ✅ 99.9% availability SLA
- ✅ Automatic scaling
- ✅ HTTPS support (with Front Door)
- ✅ Custom domain support
- ✅ CDN integration
- ✅ No server management

**Perfect For:** 🎯
- 📚 Documentation sites
- 📱 Marketing pages
- ⚛️ Single-page applications
- 📰 Static blog hosting
- 🎨 Design system showcases

### Detailed Cost Comparison with Azure Static Web Apps 📊

When choosing between static site hosting options, here's a comprehensive breakdown:

**Azure Storage + Front Door (This Tutorial)** 🏢

| Resource | Cost |
|----------|------|
| 🗄️ Storage Account (LRS) | $0.50 |
| 🌍 Front Door (Standard) | $35.00 |
| 📊 Data Transfer | $0.01-0.10 |
| **Monthly Total** | **~$36/month** |

**Benefits:** ✅
- ✅ Global CDN distribution
- ✅ Managed SSL certificates
- ✅ Custom domains with HTTPS
- ✅ Advanced caching rules
- ✅ Security headers via rules
- ✅ High availability SLA

**Use when:** 🤔
- You need global CDN performance
- Custom domain is essential
- Professional/production documentation
- Medium to high traffic sites

---

**Azure Static Web Apps (Free Alternative)** 🆓

| Resource | Cost |
|----------|------|
| 🌐 Static Web App (Free tier) | $0.00 |
| ⚡ Azure Functions (if needed) | $5-15 |
| 📊 Data Transfer | Included |
| **Monthly Total** | **~$0-15/month** |

**Benefits:**
- ✅ No baseline cost
- ✅ Git integration (auto-deploy)
- ✅ Serverless backend (optional)
- ✅ Built-in staging environments
- ✅ Custom domains supported
- ✅ Managed SSL certificates

**Limitations:**
- ❌ No CDN (slower global distribution)
- ❌ No advanced caching rules
- ❌ Limited to free tier features initially
- ❌ Cold start latency for functions

**Use when:**
- Cost is the primary concern
- Documentation accessed mainly from single region
- Lower traffic volumes
- Team/internal documentation
- Quick MVP deployment

---

**Decision Matrix**

| Factor | Storage + Front Door | Static Web Apps |
|--------|----------------------|-----------------|
| **Startup Cost** | ~$36/month | ~$0/month |
| **Global Performance** | Excellent | Good (no CDN) |
| **Setup Complexity** | Medium | Low |
| **Custom Domain** | Yes (with HTTPS) | Yes (with HTTPS) |
| **Deployment** | Pipeline-driven | Git-integrated |
| **Scaling** | Automatic | Automatic |
| **Best For** | Production docs | Quick prototypes |

---

**Cost-Benefit Analysis**

Choose **Storage + Front Door** if:
- 💰 Budget is available for production setup
- 🌍 Global audience needs fast access
- 🔒 Professional appearance important
- 📈 High traffic expected (>10GB/month)
- 🎯 Custom domain essential for branding

Choose **Static Web Apps** if:
- 💰 Minimizing costs critical
- 🌐 Primarily regional/single-region audience
- ⚡ Quick time-to-market needed
- 📊 Lower traffic (<5GB/month)
- 🏢 Internal/team documentation

---

### The Hidden Win: Consolidating Multiple Sites on One Front Door 🚀

Here's where Storage + Front Door becomes **incredibly cost-effective**: if you're already hosting other sites or APIs behind a Front Door instance, adding documentation is nearly free.

**Scenario: You Already Have Front Door** 💡

| Resource | Single Site | 2-3 Sites | 4+ Sites |
|----------|------------|----------|---------|
| 🌍 Front Door Standard | $35.00 | $35.00 | $35.00 |
| 🗄️ Storage Account #1 | $0.50 | $0.50 | $0.50 |
| 🗄️ Storage Account #2 | — | $0.50 | $0.50 |
| 🗄️ Storage Account #3 | — | $0.50 | $0.50 |
| 📊 Data Transfer | $0.10 | $0.30 | $0.50 |
| **Monthly Total** | **$35.60** | **$36.80** | **$37.50** |
| **Cost Per Site** | $35.60 | **$12.27** | **$9.38** |

**Real-World Advantage:** 💰

Once Front Door is running, each additional storage account costs only **~$0.50-1.00/month**! This is dramatically cheaper than:
- ⛔ Static Web Apps at $9-36/month per site
- ⛔ Additional App Service instances
- ⛔ Separate CDN configurations

**Multi-Site Architecture Example:** 🏗️

```
Front Door (Standard) - $35/month 🌍
├── 📚 API Documentation (Storage)
├── 📰 Engineering Blog (Storage)
├── 🎨 Design System (Storage)
├── 📊 Product Dashboards (Storage)
└── 🔧 Developer Portal (Storage)
```

**Total Cost:** ~$36-37/month for 5 sites
**Cost Per Site:** ~$7.40/month 💵

---

### When to Use This Pattern ✅

✅ **Perfect for organizations where:**
- 🌐 Multiple documentation/content sites needed
- 🔄 Front Door already deployed for API distribution
- 💰 Company paying $35/month anyway for CDN
- 🔒 Want unified SSL, caching, security policies
- 📋 Need audit trail for content delivery

✅ **Examples:** 📌
- Main API docs + Engineering blog + Design system
- Internal wiki + Public knowledge base + Marketing site
- Multiple product documentation sites
- Team/department resource portals

⚠️ **Trade-offs:** ⚠️
- 🤝 Shared Front Door instance (coordinate rules with other teams)
- 🔄 All sites use same security headers/caching policies
- 🎯 Single point of configuration (can affect multiple sites)
- 📝 May need governance around content management

---

## Architecture Overview 🏗️

Our documentation deployment pipeline looks like this:

```
┌─────────────────┐
│  Git Repository │
│   (Markdown)    │
└────────┬────────┘
         │
         │ Push triggers pipeline
         │
┌────────▼────────────┐
│  Azure DevOps       │
│  Pipeline           │
│  - Build TechDocs   │
│  - Run Tests        │
│  - Deploy to Azure  │
└────────┬────────────┘
         │
         │ Upload static files
         │
┌────────▼────────────┐
│  Azure Storage      │
│  Static Website     │
│  - HTML/CSS/JS      │
│  - Images/Assets    │
└────────┬────────────┘
         │
         │ Content distribution
         │
┌────────▼────────────┐
│  Azure Front Door   │
│  - Global CDN       │
│  - SSL/TLS          │
│  - Custom Domain    │
│  - Caching          │
└─────────────────────┘
```

## What is TechDocs? 📖

[TechDocs](https://backstage.io/docs/features/techdocs/techdocs-overview) is the documentation system built into Backstage. It converts Markdown files into beautiful, searchable documentation sites with:

- **Markdown-Based**: ✍️ Write docs in plain Markdown with your code
- **Built-in Search**: 🔍 Full-text search across all documentation
- **Component-Aware**: 🔗 Documentation tied to specific services/components
- **Beautiful UI**: 🎨 Modern, responsive design out of the box
- **Version Control**: 📦 Docs live with code, versioned together

Even if you're not using Backstage, TechDocs can generate standalone documentation sites.

## Setting Up Terraform Cloud ☁️

Before we create Azure resources, we need to set up Terraform Cloud to manage our infrastructure state and execute our deployments.

### Creating the Workspace in Terraform Cloud 🔧

1. **Log into Terraform Cloud** at https://app.terraform.io
2. **Create a new workspace**:
   - Click "New workspace"
   - Choose "CLI-driven workflow"
   - Name it `documentation`
   - Add a description: "Platform documentation infrastructure"

3. **Configure workspace settings**:
   - Go to Settings → General
   - Set Execution Mode: "Remote"
   - Set Terraform Version: "1.5.0" or later
   - Enable "Auto apply" if you want automatic deployments (optional)

### Configure Workspace Variables 🔐

In the workspace, go to Variables and add the following:

**Environment Variables** (for Azure authentication): 🔑
- `ARM_CLIENT_ID`: Your Azure service principal client ID
- `ARM_CLIENT_SECRET`: Your Azure service principal secret (**mark as sensitive**)
- `ARM_SUBSCRIPTION_ID`: Your Azure subscription ID
- `ARM_TENANT_ID`: Your Azure tenant ID

**Terraform Variables**:
- `location`: `westus2`
- `project_name`: `platformdocs`
- `custom_domain`: Leave empty or set to your custom domain (e.g., `docs.yourdomain.com`)

### Getting Azure Service Principal Credentials 🔑

If you don't have a service principal yet, create one:

```bash
# Login to Azure
az login

# Create service principal
az ad sp create-for-rbac \
  --name "sp-terraform-platform-docs" \
  --role Contributor \
  --scopes /subscriptions/{subscription-id}

# Output will show:
# {
#   "appId": "...",        # Use this for ARM_CLIENT_ID
#   "password": "...",     # Use this for ARM_CLIENT_SECRET
#   "tenant": "..."        # Use this for ARM_TENANT_ID
# }

# Get your subscription ID
az account show --query id -o tsv  # Use this for ARM_SUBSCRIPTION_ID
```

## Azure Infrastructure with Terraform

Let's build the infrastructure step by step. We'll create all the Terraform files in the `documentation/infra` folder.

## Azure Infrastructure with Terraform 🏛️

### Terraform Provider Configuration 🖥️

Create `documentation/infra/provider.tf` to configure the AzureRM provider and Terraform Cloud backend:

```hcl
```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.50"
    }
  }

  cloud {
    organization = "your-org-name"  # Replace with your Terraform Cloud organization
    
    workspaces {
      name = "documentation"
    }
  }
}
```

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
    
    key_vault {
      purge_soft_delete_on_destroy    = false
      recover_soft_deleted_key_vaults = true
    }
  }
}
```

### Resource Group and Common Resources 📦

Create `documentation/infra/main.tf` for the resource group and common tags:

```hcl
locals {
  environment = "test"  # Hardcoded for this deployment
  
  common_tags = {
    environment = local.environment
    purpose     = "platform-documentation"
    managed_by  = "terraform"
    project     = var.project_name
  }
}

resource "azurerm_resource_group" "docs" {
  name     = "rg-${var.project_name}-${local.environment}"
  location = var.location

  tags = local.common_tags
}
```

### Storage Account for Static Website 🗄️

Create `documentation/infra/storage.tf` for the storage account configured for static website hosting:

**Important Note on Storage Account Naming:** ⚠️ Azure Storage account names must be globally unique across all of Azure, 3-24 characters long, and contain only lowercase letters and numbers. In our case, we're using `st${var.project_name}` which creates `stplatformdocs` - if this name is already taken globally, you'll need to modify the `project_name` variable to make it unique.

```hcl
resource "azurerm_storage_account" "docs" {
  name                     = "st${var.project_name}"
  resource_group_name      = azurerm_resource_group.docs.name
  location                 = azurerm_resource_group.docs.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  account_kind             = "StorageV2"

  tags = local.common_tags
}

resource "azurerm_storage_account_static_website" "docs" {
  storage_account_id = azurerm_storage_account.docs.id
  index_document     = "index.html"
  error_404_document = "404.html"
}

# Output the primary endpoint
output "static_website_url" {
  value       = azurerm_storage_account.docs.primary_web_endpoint
  description = "The URL of the static website"
}

output "storage_account_name" {
  value       = azurerm_storage_account.docs.name
  description = "The name of the storage account"
}
```

### Azure Front Door for Global Distribution 🌍

Create `documentation/infra/frontdoor.tf` for Azure Front Door CDN and custom domain support:

```hcl
resource "azurerm_cdn_frontdoor_profile" "docs" {
  name                = "fd-${var.project_name}"
  resource_group_name = azurerm_resource_group.docs.name
  sku_name            = "Standard_AzureFrontDoor"

  tags = local.common_tags
}

resource "azurerm_cdn_frontdoor_endpoint" "docs" {
  name                     = "ep-${var.project_name}"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.docs.id
  
  tags = local.common_tags
}

resource "azurerm_cdn_frontdoor_origin_group" "docs" {
  name                     = "og-${var.project_name}"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.docs.id

  load_balancing {
    sample_size                 = 4
    successful_samples_required = 3
  }

  health_probe {
    path                = "/"
    request_type        = "HEAD"
    protocol            = "Https"
    interval_in_seconds = 100
  }
}

resource "azurerm_cdn_frontdoor_origin" "docs" {
  name                          = "origin-${var.project_name}"
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.docs.id

  enabled                        = true
  host_name                      = replace(replace(azurerm_storage_account.docs.primary_web_endpoint, "https://", ""), "/", "")
  http_port                      = 80
  https_port                     = 443
  origin_host_header             = replace(replace(azurerm_storage_account.docs.primary_web_endpoint, "https://", ""), "/", "")
  priority                       = 1
  weight                         = 1000
  certificate_name_check_enabled = true
}

resource "azurerm_cdn_frontdoor_route" "docs" {
  name                          = "route-${var.project_name}"
  cdn_frontdoor_endpoint_id     = azurerm_cdn_frontdoor_endpoint.docs.id
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.docs.id
  cdn_frontdoor_origin_ids      = [azurerm_cdn_frontdoor_origin.docs.id]
  cdn_frontdoor_custom_domain_ids = var.custom_domain != "" ? [azurerm_cdn_frontdoor_custom_domain.docs[0].id] : []

  supported_protocols    = ["Http", "Https"]
  patterns_to_match      = ["/*"]
  forwarding_protocol    = "HttpsOnly"
  link_to_default_domain = true
  https_redirect_enabled = true

  depends_on = [
    azurerm_cdn_frontdoor_origin.docs,
    azurerm_cdn_frontdoor_custom_domain.docs
  ]
}

output "frontdoor_endpoint" {
  value       = azurerm_cdn_frontdoor_endpoint.docs.host_name
  description = "The Front Door endpoint hostname"
}

resource "azurerm_cdn_frontdoor_custom_domain" "docs" {
  count = var.custom_domain != "" ? 1 : 0

  name                     = "custom-domain-${var.project_name}"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.docs.id
  host_name                = var.custom_domain

  tls {
    certificate_type    = "ManagedCertificate"   
  }
}
```

### Variables and Resource Group 📝

Create `documentation/infra/variables.tf` for the input variables:

```hcl
variable "project_name" {
  description = "Project name for resource naming"
  type        = string
  default     = "platformdocs"
  
  validation {
    condition     = length(var.project_name) <= 15 && can(regex("^[a-z0-9]+$", var.project_name))
    error_message = "Project name must be 15 characters or less and contain only lowercase letters and numbers."
  }
}

variable "location" {
  description = "Azure region for resources"
  type        = string
  default     = "westus2"
}

variable "custom_domain" {
  description = "Custom domain for documentation site (optional)"
  type        = string
  default     = ""
}
```

## Setting Up TechDocs 📚

TechDocs requires a specific project structure. Here's a minimal setup:

### Documentation Structure 🗂️

```
repo-root/
├── documentation/       # Main documentation folder
│   ├── mkdocs.yml      # TechDocs configuration
│   ├── docs/           # Markdown documentation files
│   │   ├── index.md    # Home page
│   │   ├── getting-started/
│   │   │   ├── index.md
│   │   │   └── quickstart.md
│   │   ├── architecture/
│   │   │   ├── index.md
│   │   │   └── decisions.md
│   │   └── api/
│   │       ├── index.md
│   │       └── reference.md
│   ├── site/           # Generated output (gitignored)
│   └── infra/          # Terraform infrastructure
│       ├── provider.tf
│       ├── main.tf
│       ├── variables.tf
│       ├── storage.tf
│       ├── frontdoor.tf
│       └── monitoring.tf
```

### MkDocs Configuration 📋

Create `documentation/mkdocs.yml`:

```yaml
site_name: Platform Documentation
site_description: Modern Development Platform Documentation
site_author: Your Team

repo_url: https://github.com/yourorg/platform-docs
edit_uri: edit/main/documentation/docs/

# Markdown files are in the docs/ subfolder
docs_dir: docs

theme:
  name: material
  palette:
    primary: indigo
    accent: indigo
  features:
    - navigation.tabs
    - navigation.sections
    - navigation.expand
    - search.suggest
    - search.highlight

nav:
  - Home: index.md
  - Getting Started:
      - Overview: getting-started/index.md
      - Quickstart: getting-started/quickstart.md
  - Architecture:
      - Overview: architecture/index.md
      - Decisions: architecture/decisions.md
  - API Reference:
      - Overview: api/index.md
      - Reference: api/reference.md

markdown_extensions:
  - admonition
  - codehilite
  - pymdownx.superfences
  - pymdownx.tabbed
  - toc:
      permalink: true

plugins:
  - search
  - techdocs-core
```

### Building TechDocs Locally

Install TechDocs CLI and test locally:

```bash
# Install TechDocs CLI
npm install -g @techdocs/cli

# Or use npx
npx @techdocs/cli

# Navigate to the documentation folder (where mkdocs.yml is located)
cd documentation

# Generate documentation (requires Docker)
techdocs-cli generate --source-dir . --output-dir ./site

# Preview locally
techdocs-cli serve --source-dir .
```

Open `http://localhost:3000` to preview your documentation.

**Troubleshooting TechDocs Generation:**

If you get a Docker error like `Docker container returned a non-zero exit code (1)`:

1. **Ensure Docker is running**: TechDocs requires Docker to be running
   ```bash
   docker ps
   ```

2. **Use the no-docker option**: Generate without Docker for local testing
   ```bash
   techdocs-cli generate --source-dir . --output-dir ./site --no-docker
   ```

3. **Install dependencies locally**: If using `--no-docker`, install Python dependencies
   ```bash
   # Install Python 3 if not already installed (macOS)
   brew install python3
   
   # Or on Ubuntu/Debian
   sudo apt-get install python3 python3-pip
   
   # Install mkdocs-techdocs-core (user-level to avoid permission issues)
   pip3 install --user mkdocs-techdocs-core
   
   # Or use a virtual environment (recommended)
   python3 -m venv venv
   source venv/bin/activate
   pip install mkdocs-techdocs-core
   ```

4. **Check mkdocs.yml**: Ensure your configuration is valid
   ```bash
   mkdocs build --strict
   ```

5. **Common mkdocs errors**:
   
   **Missing dependencies**: If you get module import errors
   ```bash
   pip install --user mkdocs-material pymdown-extensions
   ```
   
   **Invalid navigation**: Check your `nav` section in `mkdocs.yml` for typos or missing files
   ```bash
   # Test build to see specific error (run from documentation folder where mkdocs.yml is)
   cd documentation
   mkdocs build --verbose
   ```
   
   **Plugin errors**: Ensure `techdocs-core` plugin is properly installed
   ```bash
   pip show mkdocs-techdocs-core
   ```

For CI/CD pipelines, you can use the `--no-docker` flag to avoid Docker dependency:

```yaml
- script: |
    python3 -m pip install --upgrade pip
    pip3 install mkdocs-techdocs-core
    # Run from documentation folder where mkdocs.yml is located
    cd documentation
    techdocs-cli generate --source-dir . --output-dir ./site --no-docker
  displayName: 'Generate Documentation'
```

## Setting Up Azure DevOps 🔄

Before we create the pipeline, we need to set up authentication and permissions in Azure DevOps.

### Terraform Cloud Token

To create a Terraform Cloud team token:

1. Log into Terraform Cloud at https://app.terraform.io
2. Go to your organization settings
3. Click "Teams" → Select your team (or create one)
4. Click "Team API token"
5. Click "Create a team token"
6. Copy the token and save it securely
7. Use this token for the `TERRAFORM_CLOUD_TOKEN` variable in Azure DevOps

In Azure DevOps, create a pipeline variable for the Terraform Cloud token:

1. Go to Pipelines → Select your pipeline → Edit
2. Click "Variables" in the top right
3. Click "+ Add"
4. **Name**: `TERRAFORM_CLOUD_TOKEN`
5. **Value**: Your Terraform Cloud team token
6. **Keep this value secret**: ✓ Check this box
7. Click "OK"

### Service Connection

Create an Azure Resource Manager service connection in Azure DevOps:

1. Go to Project Settings → Service connections
2. Create new service connection → Azure Resource Manager
3. **Authentication method**: Select "Workload Identity federation (automatic)"
4. **Scope level**: Choose your subscription
5. **Resource group**: Leave blank (do not select a specific resource group)
6. **Service connection name**: `documentation`
7. Click "Save"

**Important:** Don't scope the service connection to a specific resource group. While this gives broader subscription-level access for control plane operations (creating/managing resources), we'll use Azure RBAC to grant specific data plane permissions in the next step.

Azure DevOps will automatically create an App Registration in your Azure AD and configure the workload identity federation for secure, keyless authentication.

### Grant Storage Permissions

The service connection created in Azure DevOps has subscription-level permissions for control plane operations (managing Azure resources like creating storage accounts, Front Door, etc.), but it needs additional data plane permissions to upload and manage blobs within the storage account.

**Control Plane vs Data Plane:**
- **Control Plane**: Managing Azure resources themselves (create, update, delete resources). The service connection has this via its Contributor role at the subscription level.
- **Data Plane**: Accessing and managing data within resources (uploading blobs, reading files, etc.). This requires separate RBAC assignments.

The service principal needs the **Storage Blob Data Owner** role to upload documentation files to the storage account:

```bash
# Get the service principal Object ID from the service connection
# (You can find this in Azure DevOps: Service Connection → Manage Service Principal → Object ID)

# Assign Storage Blob Data Owner role (data plane access)
az role assignment create \
  --assignee <service-principal-object-id> \
  --role "Storage Blob Data Owner" \
  --scope /subscriptions/<subscription-id>/resourceGroups/rg-platformdocs-test/providers/Microsoft.Storage/storageAccounts/stplatformdocstest
```

Alternatively, assign the role through the Azure Portal:

1. Navigate to your storage account in the Azure Portal
2. Click "Access Control (IAM)" in the left menu
3. Click "+ Add" → "Add role assignment"
4. Select "Storage Blob Data Owner" role
5. Click "Next"
6. Select "User, group, or service principal"
7. Click "+ Select members"
8. Search for your service connection name (`documentation`)
9. Select it and click "Select"
10. Click "Review + assign"

**Why Storage Blob Data Owner?**
- This role provides full access to blob data (read, write, delete)
- Required for the `az storage blob upload-batch` and `az storage blob delete-batch` commands in the pipeline
- Scoped to just the storage account, following least-privilege principles

## Azure DevOps Pipeline

Now let's automate the deployment with Azure DevOps.

### Pipeline Configuration

Create `azure-pipelines.yml` in your repository:

```yaml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - documentation/docs/**
      - documentation/mkdocs.yml
      - documentation/infra/**
      - .azdo/documentation.yml

pool:
  vmImage: 'ubuntu-latest'

variables:
  - name: storageAccount
    value: 'stplatformdocstest'
  - name: containerName
    value: '\$web'
  - name: TF_CLOUD_TOKEN
    value: '$(TERRAFORM_CLOUD_TOKEN)'

stages:
  - stage: Build
    displayName: 'Build Documentation'
    dependsOn: []
    jobs:
      - job: BuildDocs
        displayName: 'Build TechDocs'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '24.x'
            displayName: 'Install Node.js'

          - script: |
              npm install -g @techdocs/cli
            displayName: 'Install TechDocs CLI'

          - script: |
              cd documentation
              techdocs-cli generate --source-dir . --output-dir ./site
            displayName: 'Generate Documentation'

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: 'documentation/site'
              ArtifactName: 'documentation'
              publishLocation: 'Container'
            displayName: 'Publish Documentation Artifact'

  - stage: DeployInfrastructure
    displayName: 'Deploy Infrastructure'
    dependsOn: []
    jobs:
      - job: TerraformApply
        displayName: 'Apply Terraform'
        steps:
          - task: ms-devlabs.custom-terraform-tasks.custom-terraform-installer-task.TerraformInstaller@0
            inputs:
              terraformVersion: 'latest'
            displayName: 'Install Terraform'

          - script: |
              cat > ~/.terraformrc << EOF
              credentials "app.terraform.io" {
                token = "$TERRAFORM_CLOUD_TOKEN"
              }
              EOF
            displayName: 'Configure Terraform Cloud Credentials'
            env:
              TERRAFORM_CLOUD_TOKEN: $(TERRAFORM_CLOUD_TOKEN)

          - script: |
              cd documentation/infra
              terraform init
            displayName: 'Terraform Init'

          - script: |
              cd documentation/infra
              terraform apply -auto-approve
            displayName: 'Terraform Apply'

  - stage: Deploy
    displayName: 'Deploy to Azure'
    dependsOn: 
      - Build
      - DeployInfrastructure
    condition: succeeded()
    jobs:
      - deployment: DeployDocs
        displayName: 'Deploy to Storage'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: DownloadBuildArtifacts@0
                  inputs:
                    buildType: 'current'
                    downloadType: 'single'
                    artifactName: 'documentation'
                    downloadPath: '$(System.ArtifactsDirectory)'
                  displayName: 'Download Documentation'

                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'documentation'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      # Remove old files
                      az storage blob delete-batch --account-name $(storageAccount) --source $(containerName) --auth-mode login
                      
                      # Upload new files
                      az storage blob upload-batch --account-name $(storageAccount) --destination $(containerName) --source $(System.ArtifactsDirectory)/documentation --overwrite true --auth-mode login
                  displayName: 'Deploy to Storage Account'

                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'documentation'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      # Purge Front Door cache
                      az afd endpoint purge --resource-group rg-platformdocs-test --profile-name fd-platformdocs-test --endpoint-name ep-platformdocs-test --content-paths "/*"
                  displayName: 'Purge Front Door Cache'
```

**Key Pipeline Features:**

1. **Trigger Paths**: Pipeline runs when documentation files or the pipeline itself changes
2. **Storage Account Variable**: Set to `stplatformdocstest` - matches the storage account name created by Terraform
3. **Container Name**: Escaped as `\$web` to prevent variable expansion
4. **Terraform Cloud Token**: Uses environment variable in the script for proper credential handling
5. **Parallel Stages**: Build and DeployInfrastructure run in parallel for faster execution
6. **TerraformInstaller Task**: Uses the full task ID `ms-devlabs.custom-terraform-tasks.custom-terraform-installer-task.TerraformInstaller@0`

## Deployment Workflow 🚀

Here's the complete workflow for deploying documentation to the test environment:

### 1. Configure Azure DevOps Pipeline Variables

The pipeline variables are already set correctly:

1. `storageAccount`: `stplatformdocstest` (matches Terraform output)
2. `containerName`: `$web`
3. `TERRAFORM_CLOUD_TOKEN`: Added as a secret variable

### 2. Grant Permissions to Service Principal

```bash
# Get your service principal object ID from Azure DevOps service connection
# Then assign the role using the actual storage account name

az role assignment create \
  --assignee <service-principal-object-id> \
  --role "Storage Blob Data Owner" \
  --scope /subscriptions/<subscription-id>/resourceGroups/rg-platformdocs-test/providers/Microsoft.Storage/storageAccounts/stplatformdocstest
```

### 3. Run the Pipeline

Once configured, the pipeline will:
1. Build documentation from Markdown
2. Deploy/update infrastructure via Terraform Cloud (automatically creates resources on first run)
3. Upload to storage account
4. Purge Front Door cache

The infrastructure stage is idempotent - it won't recreate resources if they already exist. On the first pipeline run, Terraform will create all the Azure resources. On subsequent runs, it will only update resources if changes are detected.

**Note:** You don't need to run Terraform locally for deployment. The pipeline handles all infrastructure provisioning and updates through Terraform Cloud. The only time you'd run Terraform locally is to destroy resources when cleaning up (see the "Cleaning Up Resources" section below).

### Custom Domain Configuration (Optional)

To use a custom domain like `docs.yourdomain.com`:

#### Set the Custom Domain Variable

The custom domain support is already built into the `frontdoor.tf` configuration. To enable it, simply set the `custom_domain` Terraform variable in your Terraform Cloud workspace:

1. Go to your Terraform Cloud workspace → Variables
2. Add or update the `custom_domain` variable with your domain (e.g., `docs.yourdomain.com`)
3. Run the pipeline - Terraform will automatically create the custom domain resource and configure Front Door

#### DNS Configuration

After setting the custom domain variable and applying it, you need to validate domain ownership:

**Step 1: Add the custom domain CNAME**

Add a CNAME record pointing to your Front Door endpoint:

```
docs.yourdomain.com  CNAME  ep-{your-project-name}-{random-hash}.azurefd.net
```

**Step 2: Validate domain ownership**

1. Go to the Azure Portal
2. Navigate to your Front Door resource
3. Click "Domains" in the left menu
4. Find your custom domain and click on it
5. Look at the "Validation state" section
6. You'll see a TXT record that needs to be added to your DNS:
   - **Record type**: TXT
   - **Record name**: `_dnsauth.docs.yourdomain.com`
   - **Record value**: A unique validation token (e.g., `_jm6ytg2a7tnl53777awtblog2f8tylh`)

7. Add this TXT record to your DNS provider
8. Wait for DNS propagation (can take up to 15-30 minutes)
9. Azure will automatically validate the domain once the TXT record is detected
10. Once validated, Front Door will provision a managed SSL certificate

**Note:** The TXT record is only needed for validation. You can remove it after the domain is validated, but keeping it doesn't cause any issues.

Front Door will automatically provision and manage the SSL certificate for your custom domain once validation is complete.

## Monitoring and Analytics

Add monitoring to track documentation usage:

### Application Insights Integration

Create `documentation/infra/monitoring.tf` for Application Insights:

```hcl

resource "azurerm_application_insights" "docs" {
  name                = "ai-${var.project_name}-${local.environment}"
  resource_group_name = azurerm_resource_group.docs.name
  location            = azurerm_resource_group.docs.location
  application_type    = "web"

  tags = local.common_tags
}

output "instrumentation_key" {
  value       = azurerm_application_insights.docs.instrumentation_key
  sensitive   = true
  description = "Application Insights instrumentation key"
}

output "connection_string" {
  value       = azurerm_application_insights.docs.connection_string
  sensitive   = true
  description = "Application Insights connection string"
}
```

Add the Application Insights snippet to your documentation template.

## Testing the Deployment ✅

After your pipeline runs, test the deployment:

```bash
# Test the storage endpoint directly
# Replace with your actual storage account name from your project_name variable
curl https://st{your-project-name}.z5.web.core.windows.net/

# Test through Front Door
# Replace with your actual Front Door endpoint from Azure Portal or Terraform Cloud outputs
curl https://ep-{your-project-name}-{random-hash}.azurefd.net/

# Get the exact URLs from Terraform Cloud:
# 1. Log into https://app.terraform.io
# 2. Navigate to your organization → documentation workspace
# 3. Go to "States" → View latest state
# 4. Look for outputs: static_website_url and frontdoor_endpoint
# 
# Or view outputs in Azure Portal:
# - Storage endpoint: Storage Account → Static website → Primary endpoint
# - Front Door endpoint: Front Door → Endpoint hostname

# Test custom domain (if configured)
curl https://docs.yourdomain.com/
```

## Best Practices 🎯

### 1. Compression

Enable compression in Front Door for better performance:

```hcl
resource "azurerm_cdn_frontdoor_rule" "compression" {
  name                      = "compression"
  cdn_frontdoor_route_id    = azurerm_cdn_frontdoor_route.docs.id
  order                     = 1
  behavior_on_match         = "Continue"

  actions {
    response_header_action {
      header_action = "Append"
      header_name   = "Content-Encoding"
      value         = "gzip"
    }
  }

  conditions {
    request_header_condition {
      header_name  = "Accept-Encoding"
      operator     = "Contains"
      match_values = ["gzip"]
    }
  }
}
```

### 2. Security Headers

Add security headers to your documentation:

```hcl
resource "azurerm_cdn_frontdoor_rule" "security_headers" {
  name                      = "security-headers"
  cdn_frontdoor_route_id    = azurerm_cdn_frontdoor_route.docs.id
  order                     = 2
  behavior_on_match         = "Continue"

  actions {
    response_header_action {
      header_action = "Append"
      header_name   = "X-Content-Type-Options"
      value         = "nosniff"
    }

    response_header_action {
      header_action = "Append"
      header_name   = "X-Frame-Options"
      value         = "DENY"
    }

    response_header_action {
      header_action = "Append"
      header_name   = "Strict-Transport-Security"
      value         = "max-age=31536000; includeSubDomains"
    }
  }
}
```

## Troubleshooting 🔧

### Documentation Not Updating

If your documentation doesn't update after deployment:

1. **Check the pipeline**: Verify the build and deploy stages succeeded
2. **Purge the cache**: Front Door caches content aggressively
   ```bash
   az afd endpoint purge \
     --resource-group rg-platformdocs-test \
     --profile-name fd-platformdocs-test \
     --endpoint-name ep-platformdocs-test \
     --content-paths "/*"
   ```
3. **Verify upload**: Check the storage account to ensure files were uploaded
   ```bash
   az storage blob list \
     --account-name stplatformdocstest \
     --container-name '$web' \
     --output table
   ```

### 404 Errors

If you get 404 errors:

1. **Check index document**: Ensure `index.html` exists in the root
2. **Verify static website**: Confirm static website hosting is enabled
3. **Check routing**: Verify Front Door route configuration

### Custom Domain Not Working

If your custom domain doesn't work:

1. **Verify DNS**: Check CNAME record propagation
   ```bash
   nslookup docs.yourdomain.com
   ```
2. **Check certificate**: Ensure the managed certificate is provisioned (can take 15-30 minutes)
3. **Review association**: Verify the custom domain association in Front Door

## Cost Optimization 💰

Our documentation setup is already cost-effective, but you can optimize further:

### 1. Use Storage Account LRS

We're already using Locally Redundant Storage (LRS) which is the cheapest option.

### 2. Monitor Front Door Usage

Front Door Standard costs ~$35/month base rate plus:
- $0.01 per GB of data transfer
- $0.20 per million requests

For a documentation site with moderate traffic:
- 10 GB transfer/month: $0.10
- 100K requests/month: $0.02

### 3. Consider Front Door Classic

For very low traffic sites, Front Door Classic might be cheaper:
- No base rate
- Pay only for usage
- Less features, but sufficient for documentation

### Monthly Cost Breakdown

**Test/Development Setup:**
- Storage Account: $0.50
- Front Door Standard: $35.00
- Data transfer (minimal): $0.01
- **Total: ~$36/month**

**Production Setup:**
- Storage Account: $1.00 (more traffic)
- Front Door Standard: $35.00
- Data transfer (10 GB): $0.10
- **Total: ~$36/month**

**Development-Only Setup (no Front Door):**
- Storage Account only: $0.50
- Use direct storage endpoint
- **Total: ~$0.50/month**

**Note:** For this tutorial, we're deploying to a test environment with the full Front Door setup to demonstrate the complete solution. In practice, you might skip Front Door for development environments and use only the storage account endpoint to save costs.

## Cleaning Up Resources 🧹

To delete all resources and avoid charges, you'll need to run Terraform locally. This is the only time you need to run Terraform outside of the pipeline:

```bash
# Navigate to infrastructure folder
cd documentation/infra

# Login to Terraform Cloud
terraform login

# Initialize Terraform (connects to Terraform Cloud workspace)
terraform init

# Destroy all resources
terraform destroy

# Confirm the destruction
# This will remove:
# - Front Door profile and endpoints
# - Storage account and all content
# - Resource group
```

All infrastructure deployment is handled by the Azure DevOps pipeline, but cleanup requires manual intervention to prevent accidental deletion.

## Summary

We've built a complete documentation deployment pipeline that:

✅ Hosts static documentation on Azure Storage for minimal cost  
✅ Distributes globally with Azure Front Door CDN  
✅ Generates beautiful docs with TechDocs  
✅ Automates deployment with Azure DevOps pipelines  
✅ Manages infrastructure with Terraform Cloud  
✅ Supports custom domains with managed SSL certificates  
✅ Deploys to test environment (~$36/month) with same infrastructure as production  

**Key Takeaways:**
- Azure Storage account names must be globally unique - choose a unique project name
- The storage account name is created from `st${var.project_name}` - in this example `stplatformdocs` from project_name "platformdocs"
- Infrastructure and deployment are fully automated through Terraform Cloud and Azure DevOps
- Same pattern works for dev, test, and production - just change the environment in main.tf
- Front Door adds $35/month but provides global CDN, SSL, and custom domains

This infrastructure gives us enterprise-grade documentation hosting at a fraction of the cost of traditional web hosting, with the reliability and performance of Azure's global infrastructure.

The pattern we've built here can be used for any static site - documentation, marketing pages, SPAs, or blogs. The combination of Azure Storage, Front Door, and automated CI/CD creates a robust, scalable, and cost-effective hosting solution.
